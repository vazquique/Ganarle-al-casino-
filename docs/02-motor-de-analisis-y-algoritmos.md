# 02 — Motor de análisis y algoritmos

El `pricing-engine` transforma features en **distribuciones de probabilidad
calibradas y coherentes entre mercados**, y el `value-detector` las contrasta con
los precios disponibles. Están deliberadamente separados: el primero no sabe qué
cuotas existen (evita el sesgo de "encajar" el modelo al mercado), el segundo no
sabe cómo se calculó la probabilidad.

```
Features (point-in-time)
      │
      ├─► [1] Modelo generativo    ──┐
      ├─► [2] Modelo ML tabular    ──┤
      └─► [3] Consenso de mercado  ──┤
                                     ▼
                          [4] Blending + Calibración
                                     │
                                     ▼
                      p̂ con intervalo creíble [p_lo, p_hi]
                                     │
                                     ▼
                     [5] Detector de valor y filtros de ruido
                                     │
                                     ▼
                  [6] Dimensionamiento (Kelly) → Señal publicada
```

---

## 2.1 Paso 0 — De cuota a probabilidad: el *devigging*

Antes de comparar nada hay que retirar el margen. Hacerlo mal introduce un sesgo
sistemático mayor que el edge que se busca.

Dado el conjunto de cuotas `{o_i}` de un mercado, el *overround* es
`Σ 1/o_i = 1 + m`. Métodos implementados:

| Método | Fórmula | Cuándo usarlo |
|---|---|---|
| **Multiplicativo** (proporcional) | `p_i = (1/o_i) / Σ(1/o_j)` | Nunca como única opción: asume margen proporcional y **sobrestima sistemáticamente los favoritos** |
| **Aditivo** | `p_i = 1/o_i − m/n` | Mercados de 2 vías equilibrados |
| **Shin (1992)** | Resuelve `z` tal que `p_i = [√(z² + 4(1−z)·(1/o_i)²/Σ) − z] / (2(1−z))` | **Recomendado por defecto.** Modela la proporción `z` de apostantes informados; captura el *favourite–longshot bias* |
| **Potencia / Odds-ratio** (Wisdom of the Crowd, Joseph Buchdahl) | `p_i = (1/o_i)^k`, resolviendo `k` tal que `Σ p_i = 1` | Alternativa robusta; benchmark contra Shin |
| **Logarítmico** | `p_i ∝ (1/o_i)^(1/τ)` | Mercados de muchas vías (marcador exacto, goleador) |

- `RF-P-01` El método de devigging es **configurable por mercado y validado
  empíricamente**: se selecciona el que minimiza el log-loss sobre resultados
  históricos reales de ese mercado, no por preferencia teórica.
- `RF-P-02` En mercados de muchas vías (marcador exacto, primer goleador), donde
  el margen se concentra desproporcionadamente en las selecciones largas, es
  obligatorio el método logarítmico o de potencia.
- `RF-P-03` Para Betfair no se hace devigging: se usa el punto medio back/lay
  ponderado por volumen, corregido por la comisión efectiva del usuario.

---

## 2.2 Capa 1 — Modelo generativo del partido

La pieza que garantiza **coherencia entre mercados**. En lugar de entrenar un
clasificador por mercado (que produce inconsistencias absurdas), se simula el
partido y se derivan todos los mercados de la misma simulación.

### 2.2.1 Base: Dixon-Coles con decaimiento temporal

Punto de partida clásico y sorprendentemente competitivo:

```
λ_local    = exp(α_local + β_visitante + γ_campo)
μ_visitante = exp(α_visitante + β_local)

P(X=x, Y=y) = τ(x, y, λ, μ) · Poisson(x; λ) · Poisson(y; μ)
```

donde `τ` es la corrección de Dixon-Coles para marcadores bajos (0-0, 1-0, 0-1,
1-1), donde el Poisson independiente falla de forma conocida. Los parámetros se
estiman por máxima verosimilitud ponderada:

```
L(θ) = Σ_t  φ(t) · log P(resultado_t | θ),     φ(t) = exp(−ξ · Δt)
```

con semivida `ln(2)/ξ` calibrada por competición (típicamente 6–18 meses;
se ajusta por validación, no por intuición).

### 2.2.2 Producción: modelo bayesiano jerárquico sobre xG

El modelo de producción sustituye goles por **xG ajustado**, reduciendo
sustancialmente la varianza de la señal de entrada, y añade estructura
jerárquica (Baio & Blangiardo, 2010) con *shrinkage* hacia la media de la
competición:

```stan
// Esqueleto del modelo (Stan / NumPyro)
parameters {
  vector[T] att_raw;          // fuerza ofensiva por equipo
  vector[T] def_raw;          // fuerza defensiva por equipo
  real<lower=0> sigma_att;
  real<lower=0> sigma_def;
  vector[T] home_adv;         // ventaja de campo POR EQUIPO, no global
  real<lower=0> phi;          // sobredispersión (Binomial Negativa)
}
transformed parameters {
  vector[T] att = att_raw - mean(att_raw);   // identificabilidad: suma cero
  vector[T] def = def_raw - mean(def_raw);
}
model {
  sigma_att ~ exponential(1);
  sigma_def ~ exponential(1);
  att_raw   ~ normal(0, sigma_att);
  def_raw   ~ normal(0, sigma_def);
  home_adv  ~ normal(mu_home, sigma_home);   // jerárquico

  for (n in 1:N) {
    real log_lambda = att[home[n]] + def[away[n]] + home_adv[home[n]] + X[n] * beta;
    real log_mu     = att[away[n]] + def[home[n]]                     + X[n] * beta_a;
    xg_home[n] ~ neg_binomial_2_log(log_lambda, phi);
    xg_away[n] ~ neg_binomial_2_log(log_mu, phi);
  }
}
```

Ventajas decisivas sobre el MLE puntual:
- Los equipos con pocos partidos (ascendidos, inicio de temporada) se contraen
  automáticamente hacia la media — sin heurísticas ad hoc.
- La **distribución posterior** proporciona directamente el intervalo creíble de
  cada probabilidad, que es el insumo del umbral dinámico del detector de valor.
- La **ventaja de campo por equipo** captura efectos reales (altitud, ambiente,
  dimensiones del campo) que un `γ` global promedia y destruye.

**Covariables `X`** inyectadas en el predictor lineal: días de descanso, viaje,
delta de fuerza por ausencias, viento, temperatura, importancia del partido.

### 2.2.3 Simulación Monte Carlo a nivel de disparo

Del modelo de tasas a la distribución de resultados:

1. Muestrear `(λ, μ)` de la posterior — **no usar la media posterior**, eso
   colapsa la incertidumbre de parámetros y produce sobreconfianza.
2. Muestrear número de disparos por equipo (Binomial Negativa).
3. Asignar a cada disparo una calidad `xG_i` de la distribución empírica de
   calidad del equipo (mezcla de jugada abierta / balón parado / contragolpe).
4. Resolver cada disparo como Bernoulli(`xG_i`) ajustado por la capacidad de
   parada del portero rival (`PSxG−GA` con shrinkage).
5. Simular córners y tarjetas como procesos acoplados al estado del partido
   (un equipo por detrás en el minuto 80 dispara y comete más faltas — la
   dependencia del marcador es esencial y es donde fallan los modelos ingenuos).
6. Repetir `N = 50.000` iteraciones → matriz completa de marcadores exactos.

De esa única matriz se derivan, por construcción coherentes: 1X2, doble
oportunidad, O/U para toda línea, hándicap asiático y europeo, BTTS, marcador
exacto, mitad/final, córners, tarjetas, y combinaciones intra-partido con su
**correlación real** (cosa que ningún combinador multiplicativo hace bien).

- `RF-P-04` La simulación es determinista dada una semilla; `sim_seed` se
  persiste con cada precio para reproducibilidad total.
- `RF-P-05` Error de Monte Carlo objetivo: `< 0,2 %` en probabilidad para
  mercados principales. Con N = 50.000, `σ_MC ≈ √(p(1−p)/N) ≈ 0,22 %` en p=0,5 →
  se eleva N a 200.000 para mercados de margen fino.

---

## 2.3 Capa 2 — Modelo de aprendizaje automático

Complementa al generativo capturando no linealidades e interacciones que el
modelo estructural no representa.

**Algoritmo:** LightGBM con objetivo `multiclass` (1/X/2) y `regression` para
totales, más CatBoost como challenger. *No* redes profundas en v1: con
~50.000 partidos etiquetados y features tabulares, el gradient boosting domina
y es interpretable vía SHAP.

**Familias de features (≈ 180 en v1):**

| Familia | Ejemplos |
|---|---|
| Fuerza | Elo/Glicko-2, ratings att/def posteriores del modelo bayesiano |
| Forma | EWMA de npxG a favor/en contra (semividas 5/10/20), ajustada por rival |
| Estilo | PPDA, field tilt, % balón parado, ritmo (posesiones/90) |
| Plantilla | Δ fuerza por ausencias, minutos acumulados 14 días, edad media XI |
| Contexto | Descanso, viaje (km), altitud, importancia, competición paralela |
| Entorno | Viento, lluvia, temperatura, superficie, techo |
| Árbitro | Tarjetas/90 y penaltis/90 con shrinkage |
| **Mercado** | Probabilidad devigged de Pinnacle, deriva desde apertura, dispersión entre libros, volumen Betfair |

> **La familia "Mercado" es deliberadamente la más predictiva.** Excluirla para
> "no contaminar el modelo" es un error frecuente: produce un modelo peor que
> encuentra "valor" en todas partes. Se incluye, y la aportación propia se mide
> como *mejora incremental sobre el modelo que solo usa mercado* — que es la
> única definición honesta de edge.

**Higiene obligatoria:**
- `RF-P-06` Todas las features se materializan mediante *point-in-time join*
  contra el feature store. Ningún valor posterior a `t` puede entrar en el
  vector de `t`.
- `RF-P-07` Validación con **Purged K-Fold con embargo** (López de Prado):
  bloques temporales contiguos, purga de solapamientos y embargo de 7 días entre
  train y test para eliminar fuga por partidos correlacionados (misma jornada,
  mismo equipo).
- `RF-P-08` Prohibido el `KFold` aleatorio y prohibida la normalización sobre el
  dataset completo. Ambos inflan artificialmente el rendimiento del backtest.

---

## 2.4 Capa 3 — Calibración

Un modelo con 60 % de acierto y mala calibración es inútil para apostar; uno con
50 % de acierto y calibración perfecta es explotable. **La calibración es el
requisito, no la precisión.**

| Técnica | Aplicación |
|---|---|
| **Calibración de Dirichlet** | Multiclase (1X2). Preferida: preserva la restricción del símplex |
| **Regresión isotónica** | Mercados binarios con `n > 5.000` |
| **Platt scaling / temperature** | Binarios con `n` moderado; menos flexible, menos sobreajustable |
| **Venn-ABERS** | Cuando se necesita un intervalo de calibración, no un punto |

- `RF-P-09` El calibrador se ajusta **en un conjunto de calibración temporalmente
  posterior al de entrenamiento y anterior al de test.** Calibrar sobre el mismo
  fold de entrenamiento invalida la medición.
- `RF-P-10` Criterio de aceptación: **ECE (Expected Calibration Error) < 0,02**
  en 10 bins, y diagrama de fiabilidad dentro de la banda de confianza del 95 %
  en todos los bins con `n ≥ 100`.
- `RF-P-11` Monitorización continua de calibración en producción con ventana
  deslizante de 500 eventos; degradación → alerta y paso a modo sombra.

---

## 2.5 Capa 4 — *Blending*: la decisión más importante del sistema

Combinar la estimación propia con la referencia de mercado. Formulación en
espacio logit:

```
logit(p̂) = w_m · logit(p_mercado) + w_g · logit(p_generativo) + w_ml · logit(p_ml) + b
```

- Los pesos `w` se aprenden por **stacking** con regresión logística regularizada
  sobre un conjunto de validación temporalmente separado.
- Los pesos son **específicos por competición, mercado y ventana temporal**
  (T−72 h vs T−1 h). En Premier League a T−1 h el mercado domina (`w_m ≈ 0,80`);
  en segunda división noruega a T−48 h la aportación propia es mucho mayor.
- El mercado actúa como *prior* bayesiano: la aportación propia solo desplaza la
  estimación en proporción a la evidencia que aporta.

> **Consecuencia directa y contraintuitiva:** con `w_m = 0,80`, para generar un
> edge del 3 % el modelo propio debe discrepar del mercado en ~15 puntos
> porcentuales. Esto es *correcto y deseado*: filtra automáticamente el 95 % de
> las falsas señales que produce un sistema que compara el modelo crudo contra
> la cuota. La mayoría de "value bets" publicadas por herramientas comerciales
> son exactamente este artefacto.

**Excepción — modo "noticia".** Cuando el sistema detecta información que el
mercado aún no ha incorporado (alineación confirmada, `first_seen_at` anterior
al último tick de cuota de la casa objetivo), `w_m` se reduce temporalmente:
el precio de mercado es *conocidamente obsoleto* y no debe pesar. Esta es la
señal de mayor valor esperado del producto.

---

## 2.6 Capa 5 — Detector de valor

### 2.6.1 Cálculo base

```python
edge   = p_hat * odds - 1.0            # valor esperado por unidad apostada
ev_pct = edge * 100

# Incertidumbre propagada de la posterior + error MC + error de calibración
sigma_p    = sqrt(var_posterior + var_mc + var_calibration)
sigma_edge = odds * sigma_p
z          = edge / sigma_edge
```

### 2.6.2 Condiciones de publicación (todas obligatorias)

| # | Condición | Umbral por defecto | Motivo |
|---|---|---|---|
| C1 | `edge > edge_min(mercado)` | 2,0 % principales / 3,5 % nicho | Cubrir el error residual del modelo |
| C2 | `z > z_min` | 1,64 (≈ 95 % unilateral) | Significancia estadística real |
| C3 | `p̂` dentro del rango de calibración validado | `0,05 ≤ p̂ ≤ 0,95` | Fuera de rango el calibrador extrapola |
| C4 | Overround del libro objetivo < umbral | < 8 % (3 vías) | Un libro con 12 % de margen y "valor" está mal leído |
| C5 | Liquidez / límite estimado | ≥ 25 € ejecutables | Una señal no ejecutable no es una señal |
| C6 | Ausencia de contradicción de mercado | ver 2.6.3 | Filtro anti-trampa |
| C7 | Frescura del precio | < 30 s | Evitar perseguir cuotas muertas |
| C8 | Datos completos del fixture | sin `BLOCKED_*` | Integridad |

### 2.6.3 Filtros anti-ruido y anti-trampa

Esta sección es la que separa un producto profesional de un generador de ruido.

1. **Contradicción de mercado.** Si el modelo ve valor en el local pero los
   libros sharp llevan 10 minutos moviéndose *contra* el local, la hipótesis más
   probable es que el mercado sabe algo que nosotros no (alineación filtrada,
   lesión en calentamiento). **Se suprime la señal**, no se refuerza.
2. **Cuota atípica aislada (*outlier*).** Una casa a 3,40 cuando las otras 18
   están entre 2,70 y 2,85 no suele ser valor: suele ser un error que será
   anulado, o una línea que ya no existe. Se marca como `SUSPECT_OUTLIER`,
   requiere confirmación con un segundo libro y se muestra con aviso explícito.
3. **Test de Kolmogorov-Smirnov sobre el residuo del segmento.** Si en el
   segmento (liga × mercado × rango de cuota) la distribución de resultados se
   desvía de lo predicho, el segmento se desactiva automáticamente.
4. **Control de multiplicidad.** Evaluando ~15.000 combinaciones
   evento×mercado×selección×casa al día, con `α = 0,05` aparecerían ~750 falsos
   positivos diarios por azar puro. Se aplica **Benjamini-Hochberg** sobre las
   señales candidatas del día para controlar el FDR al 10 %, y el `z_min`
   efectivo se ajusta en consecuencia.
5. **Circuit breaker por segmento.** Si un segmento acumula CLV medio negativo
   en las últimas 200 señales, se suspende automáticamente y se notifica al
   equipo. La decisión de reactivar es humana.
6. **Supresión de correlación.** Múltiples señales del mismo partido se agrupan
   en una sola recomendación con exposición conjunta (ver
   [§04.3](04-gestion-de-riesgo-y-bankroll.md#43-kelly-con-apuestas-correlacionadas)).

### 2.6.4 Clasificación de confianza mostrada al usuario

| Tier | Criterio | Presentación |
|---|---|---|
| **A** | `z > 2,33` (99 %), segmento con ≥ 500 señales históricas y CLV > 0, liquidez alta | Verde, alerta push |
| **B** | `z > 1,64`, segmento con ≥ 200 señales, liquidez media | Ámbar, alerta en app |
| **C** | Cumple C1–C8 pero segmento inmaduro (`n < 200`) | Gris, solo visible con filtro explícito, **excluido del staking automático** |

---

## 2.7 Modelos específicos por mercado

| Mercado | Enfoque |
|---|---|
| **1X2 / Doble oportunidad** | Derivado de la matriz de marcadores |
| **O/U y Hándicap asiático** | Derivado de la matriz; cuartos de línea (`-0,25`) se descomponen en dos medias apuestas |
| **Córners** | Proceso de Poisson acoplado a *field tilt* y al estado del marcador; fuerte sensibilidad al viento |
| **Tarjetas** | Binomial Negativa con perfil del árbitro (shrinkage), rivalidad histórica e importancia del partido. **El mercado de mayor margen y menor atención sharp** |
| **Props de jugador** | Modelo en dos etapas: P(minutos ≥ umbral) × distribución condicional de la acción. Alta varianza: exige tier de confianza más estricto |
| **Marcador exacto** | De la matriz, pero devigging logarítmico obligatorio; márgenes del 15–25 % hacen que casi nunca haya valor real |

---

## 2.8 Validación y *backtesting*

### 2.8.1 Métricas de modelo

| Métrica | Objetivo v1.0 | Nota |
|---|---|---|
| **RPS** (Rank Probability Score) — 1X2 | ≤ 0,190 | Estándar del dominio; respeta el orden natural de los resultados. La línea de cierre de Pinnacle ronda 0,185–0,195 en ligas top |
| **Log-loss** multiclase | ≤ 0,98 | |
| **Brier score** binarios | ≤ 0,215 | |
| **ECE** | < 0,02 | Requisito bloqueante |
| **Mejora sobre "solo mercado"** | ΔRPS ≥ 0,002 | La única prueba de aportación propia |

### 2.8.2 Métrica primaria de estrategia: **CLV**

```
CLV = (cuota_obtenida / cuota_justa_de_cierre) − 1
```

donde `cuota_justa_de_cierre` es la línea de cierre sin margen (Shin sobre
Pinnacle, o Betfair al inicio).

**Por qué el CLV y no el ROI:** con un yield real del 3 % y desviación típica de
~1,0 unidad por apuesta, se necesitan **> 4.000 apuestas** para distinguir
estadísticamente un yield de +3 % de uno de 0 %. El CLV converge en cientos de
apuestas porque compara contra un estimador de baja varianza en lugar de contra
un resultado binario. Un sistema con CLV medio positivo y sostenido es rentable
a largo plazo casi por construcción; uno con ROI positivo y CLV negativo está
teniendo suerte y lo perderá.

| KPI de estrategia | Objetivo |
|---|---|
| CLV medio por señal publicada | **> +1,5 %** |
| % de señales con CLV > 0 | **> 55 %** |
| t-stat del CLV medio (n ≥ 1.000) | **> 3,0** |
| Yield simulado (Kelly 1/4) | > +2 % (informativo, no bloqueante) |
| Max drawdown simulado (p95) | < 25 % de la bankroll |

### 2.8.3 Protocolo de backtest

- **Walk-forward** con reentrenamiento mensual: entrena hasta `t`, predice
  `[t, t+1 mes)`, avanza. Nunca un único split.
- **Precios reales y ejecutables**: la cuota usada es la que estaba publicada en
  el instante de la señal, **no** la mejor cuota del día ni la de cierre.
- **Coste de ejecución modelado**: deslizamiento (la cuota se mueve entre la
  alerta y el clic), rechazo parcial de apuesta, comisión de exchange (2–5 %),
  y tasa de anulación por *palp*.
- **Simulación de límites**: aplicar los límites máximos realistas de cada casa;
  un backtest que asume stake ilimitado a cuotas de nicho es ficción.
- **Sin conocimiento del futuro en ningún punto:** el pronóstico meteorológico
  usado es el vigente en `t`, las lesiones las conocidas en `t`, los ratings
  los estimados con datos hasta `t`.

### 2.8.4 Protección contra el sobreajuste de estrategia

Probar 300 configuraciones y publicar la mejor es *data snooping*, y es la razón
por la que casi todos los sistemas de apuestas funcionan en backtest y fracasan
en vivo. Contramedidas obligatorias:

- Registro de **todas** las configuraciones evaluadas en MLflow (no solo la
  ganadora).
- **Deflated Sharpe Ratio** / prueba de White (*Reality Check*) para corregir el
  número de pruebas realizadas.
- **Hold-out final sellado**: los últimos 12 meses de datos permanecen cifrados
  y solo se descifran una vez, para la validación de aceptación previa al
  lanzamiento. Si falla, no se reajusta el modelo contra ellos — se vuelve a
  diseño con un nuevo hold-out.
- **Paper trading obligatorio**: 8 semanas en producción sin dinero real antes de
  publicar señales a usuarios. El criterio de paso es el CLV, no el ROI.

---

## 2.9 MLOps

| Componente | Herramienta | Función |
|---|---|---|
| Feature store | **Feast** (Redis online + Parquet/Iceberg offline) | Garantiza consistencia train/serve y *point-in-time joins* |
| Registro de modelos | **MLflow** | Versionado, linaje, métricas, artefactos, promoción de estados |
| Orquestación | **Dagster** | DAGs con activos versionados, *backfills* y verificación de frescura |
| Serving | **BentoML** / FastAPI + ONNX | Inferencia de baja latencia |
| Monitorización de deriva | **Evidently AI** | PSI/KS sobre features, deriva de predicción, calibración móvil |
| Experimentación | Champion/Challenger + *shadow mode* | Ningún modelo llega a usuarios sin 4 semanas en sombra |

**Política de reentrenamiento:**
- Ratings bayesianos: actualización incremental diaria (variational inference)
  y refit completo (MCMC/NUTS) semanal.
- LightGBM: reentrenamiento semanal, promoción automática solo si mejora RPS y
  ECE en el fold de validación más reciente.
- Calibradores: refit semanal con ventana de 12 meses.
- **Kill switch** automático: deriva de calibración `ECE > 0,05` sostenida
  durante 200 eventos → el modelo deja de publicar señales y el sistema alerta.
