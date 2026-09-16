# 07 — Plan de entrega, KPIs y cumplimiento

## 7.1 Fases de entrega

El plan está deliberadamente ordenado para **matar el proyecto barato si la
hipótesis de edge no se sostiene**. No se construye producto antes de demostrar
señal.

### Fase 0 — Validación de la hipótesis (6–8 semanas, 2 personas)

**Nada de interfaz. Nada de infraestructura de producción.** Cuadernos,
Parquet y un backtest riguroso.

| Entregable | Detalle |
|---|---|
| Histórico cargado | 10 temporadas × 6 ligas: resultados, xG, cuotas de apertura y cierre (football-data.co.uk + Understat + API-Football) |
| Resolución de entidades | Mapeo de equipos entre las 3 fuentes, con verificación manual completa |
| Modelo base | Dixon-Coles con decaimiento temporal + devigging de Shin |
| Backtest | Walk-forward, purged K-fold, precios reales de entrada, costes modelados |
| Informe de decisión | RPS, ECE y **CLV simulado** frente a la línea de cierre |

**Puerta de decisión (go/no-go):** `RPS ≤ 0,195` **y** `ECE < 0,03` **y**
`CLV medio > +0,5 %` sobre ≥ 3.000 apuestas simuladas. Si no se alcanza, se
itera el modelo o **se detiene el proyecto**. Construir producto sobre un edge
no demostrado es la forma más cara de fracasar en este dominio.

### Fase 1 — MVP interno (10–12 semanas, 4 personas)

- Ingesta en vivo: The Odds API + Pinnacle + API-Football + Visual Crossing.
- Pipeline canónico, `entity-resolver` con cola de revisión, TimescaleDB.
- `pricing-engine` v1 (bayesiano jerárquico + Monte Carlo) y `value-detector`
  con los ocho filtros de [§02.6.2](02-motor-de-analisis-y-algoritmos.md#262-condiciones-de-publicación-todas-obligatorias).
- Value Board web, tracker manual, cálculo automático de CLV.
- **8 semanas de paper trading obligatorio** con el equipo interno.

**Puerta:** CLV real medido > +1,0 % sobre ≥ 500 señales, con `t > 2,0`.

### Fase 2 — Beta cerrada (10 semanas, 6 personas)

- 50–100 usuarios seleccionados, con modo paper obligatorio los primeros 30 días.
- Módulo completo de bankroll: Kelly bayesiano, Kelly conjunto, límites, tilt.
- Alertas push y Telegram con control de fatiga.
- Mapas de calor, explicabilidad SHAP, simulador de ruina.
- Ampliación a 12 competiciones y a mercados de córners y tarjetas.
- Capa ML (LightGBM) y *blending* aprendido, promocionada solo tras 4 semanas en
  modo sombra.

**Puerta:** CLV > +1,5 % sobre ≥ 2.000 señales; retención a 30 días > 55 %;
cero incidentes de integridad de datos con impacto en señal.

### Fase 3 — Lanzamiento y expansión (continuo)

- Apertura comercial, planes de suscripción, app móvil nativa.
- Tenis y baloncesto (modelos punto a punto y de posesiones).
- Integración opcional con API de exchange (Betfair) para colocación asistida.
- Exploración de *in-play* con datos de tracking — proyecto independiente, con
  su propia validación de hipótesis.

---

## 7.2 Criterios de aceptación del producto

### 7.2.1 Bloqueantes (sin ellos no se lanza)

| # | Criterio | Umbral |
|---|---|---|
| A1 | Calibración del modelo principal | ECE < 0,02 |
| A2 | RPS en 1X2, ligas en alcance | ≤ 0,190 |
| A3 | Aportación sobre "solo mercado" | ΔRPS ≥ 0,002, `p < 0,05` |
| A4 | CLV medio de señales publicadas | > +1,5 %, `n ≥ 1.000`, `t > 3,0` |
| A5 | Señales con CLV positivo | > 55 % |
| A6 | Latencia tick → alerta (p95) | < 800 ms |
| A7 | Cobertura de fixtures en alcance | ≥ 95 % |
| A8 | Resolución de entidades sin error | 100 % verificado; 0 inversiones local/visitante |
| A9 | Reproducibilidad del backtest | Determinista dado `run_id` |
| A10 | Controles de juego responsable | Todos los `RF-BR-09..11` implementados y auditados |
| A11 | RGPD | Exportación y supresión funcionales, cifrado verificado |
| A12 | Ausencia de *lookahead* | Test de CI train/serve en verde sobre 1.000 fixtures |

### 7.2.2 De producto

| Métrica | Objetivo a 6 meses |
|---|---|
| Retención a 30 días | > 55 % |
| Usuarios que configuran límites de bankroll | > 70 % |
| Señales accionadas sobre publicadas (tier A) | > 35 % |
| Usuarios con CLV positivo a 100 apuestas | > 60 % |
| Ratio de sobre-stake (stake > 2× sugerido) | < 8 % de las apuestas |
| Usuarios que activan enfriamiento voluntario | Métrica de salud, sin objetivo — se monitoriza, no se optimiza |

> La última fila es intencional: **no todas las métricas deben maximizarse.** Un
> producto que optimiza el tiempo en app o el número de apuestas está optimizando
> el daño al usuario.

---

## 7.3 Equipo y costes

| Rol | FTE F1 | FTE F2 | Responsabilidad |
|---|---|---|---|
| Ingeniero de datos | 1 | 1,5 | Ingesta, resolución de entidades, calidad |
| Científico de datos / modelización | 1 | 2 | Modelos, calibración, backtest |
| Ingeniero backend | 1 | 1,5 | Servicios, API, latencia |
| Ingeniero frontend | 1 | 1 | Web, PWA, visualización |
| SRE / plataforma | 0,5 | 1 | K8s, observabilidad, coste |
| Producto / diseño | 0,5 | 1 | UX, investigación, juego responsable |
| **Total** | **5** | **8** | |

**Coste operativo mensual estimado (Fase 2):**

| Partida | Coste |
|---|---|
| Datos (cuotas, estadísticas, clima) | 600–1.100 € |
| Infraestructura cloud (EKS, RDS/Timescale, ClickHouse, S3, Redpanda) | 1.200–2.000 € |
| Observabilidad y herramientas SaaS | 300 € |
| **Total infraestructura + datos** | **≈ 2.100–3.400 €/mes** |
| Personal (8 FTE, coste empresa) | 45.000–60.000 €/mes |

Con proveedor de eventos Nivel 1 (Opta/StatsBomb) el coste de datos sube a
2.500–8.000 €/mes; la decisión se toma solo si la Fase 2 demuestra que el
cuello de botella del edge es la calidad del dato y no la modelización.

---

## 7.4 Riesgos del proyecto

| Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|
| **El edge no existe a escala explotable** | Media | Crítico | Fase 0 con puerta go/no-go antes de construir producto |
| **Overfitting del backtest** | Alta | Crítico | Hold-out sellado, Deflated Sharpe, paper trading de 8 semanas |
| **Error de resolución de entidades** | Alta | Alto | Umbral de confianza, cola humana, test anti-inversión, bloqueo de fixture |
| **Limitación masiva de cuentas de usuarios** | **Muy alta** | Alto | Gestión de límites, diversificación, ruta a exchange, expectativas explícitas desde el registro |
| Cambio de precios o de términos de un proveedor | Media | Medio | Abstracción por adaptador; dos proveedores por familia crítica |
| Degradación del modelo por cambio de régimen (reglas, VAR, formato de competición) | Media | Medio | Monitorización de calibración, CUSUM de CLV, kill switch |
| Cambio regulatorio sobre servicios de pronósticos | Media | Alto | Ver §7.5; diseño sin afiliación ni publicidad de operadores |
| Latencia insuficiente frente a competidores | Media | Medio | Presupuesto de latencia medido desde el día 1; foco en señales por noticia, no solo en velocidad pura |
| Percepción de "sistema infalible" por parte del usuario | Alta | Alto | Intervalos siempre visibles, simulador de ruina, comunicación de varianza |

---

## 7.5 Marco legal y cumplimiento

> Esta sección enumera obligaciones de diseño. **No sustituye al asesoramiento
> jurídico**, que debe obtenerse por jurisdicción antes del lanzamiento
> comercial.

### 7.5.1 Naturaleza del servicio

El producto es un **servicio de información y análisis estadístico**, no un
operador de juego: no acepta ni cursa apuestas, no custodia fondos de usuarios
ni ofrece premios. En España, por tanto, **no requiere licencia de la DGOJ** al
amparo de la Ley 13/2011 de regulación del juego. Esta clasificación es
frágil y debe preservarse activamente:

- `RC-01` **Nunca** aceptar depósitos, custodiar fondos ni liquidar apuestas.
- `RC-02` **Nunca** actuar como intermediario en la colocación de apuestas en
  nombre del usuario en casas blandas.
- `RC-03` La integración opcional con exchange se ejecuta **con las credenciales
  de API del propio usuario, sobre su propia cuenta**, y es siempre revocable.

### 7.5.2 Publicidad y afiliación

El Real Decreto 958/2020 (comunicaciones comerciales de actividades de juego en
España) restringe severamente la publicidad de juego y alcanza a los afiliados.

- `RC-04` **El modelo de negocio es la suscripción, no la afiliación.** No se
  perciben comisiones de operadores por usuario referido. Esto elimina el
  conflicto de interés estructural —recomendar más volumen de apuesta para
  cobrar más comisión— y reduce drásticamente la exposición regulatoria.
- `RC-05` Los enlaces a casas de apuestas son **funcionales y neutros** (abrir
  el mercado correspondiente), sin material promocional, sin bonos y sin
  ordenación por remuneración. El orden de casas responde exclusivamente a la
  mejor cuota disponible.
- `RC-06` Prohibida toda afirmación de rentabilidad garantizada, "sistema
  infalible" o similar, en producto y en marketing. Además de ser falso, sería
  publicidad engañosa (Ley 3/1991 de Competencia Desleal y normativa de
  consumo).

### 7.5.3 Protección del usuario

- `RC-07` Verificación de edad (18+) y bloqueo por geolocalización en
  jurisdicciones no soportadas.
- `RC-08` Herramientas de autoexclusión y enfriamiento con efecto inmediato e
  irrevocable durante su vigencia ([§04.6.2](04-gestion-de-riesgo-y-bankroll.md#462-límites-autoimpuestos-y-periodos-de-enfriamiento)).
- `RC-09` Enlace permanente y visible a recursos de ayuda según jurisdicción
  (FEJAR y Juego Seguro en España; GamCare / BeGambleAware en Reino Unido).
- `RC-10` Aviso de riesgo en el registro y en el primer uso, con reconocimiento
  explícito del usuario.

### 7.5.4 Datos

- `RC-11` **RGPD / LOPDGDD**: base legal documentada, minimización, cifrado en
  reposo de datos financieros y de apuestas, retención de 24 meses,
  portabilidad y supresión en < 30 días. La actividad de apuestas es dato
  sensible de facto por su potencial de perjuicio, y se trata en consecuencia.
- `RC-12` **Licencias de terceros**: los contratos de Opta, Sportradar y
  StatsBomb prohíben la redistribución de datos crudos. El producto expone
  **derivados** (probabilidades, señales, agregados), nunca el feed original.
  Auditoría contractual previa a cada integración.
- `RC-13` **Scraping**: revisión de los términos de servicio de cada fuente
  abierta antes de su uso en producción; sustitución por fuente licenciada
  cuando el uso comercial no esté permitido.
- `RC-14` **Reglamento (UE) 2024/1689 de IA**: el sistema no entra en las
  categorías de alto riesgo del Anexo III, pero se aplican por diseño las
  obligaciones de transparencia: el usuario sabe que las recomendaciones son
  generadas por modelos estadísticos, con qué incertidumbre y con qué factores
  ([§03.5](03-interfaz-y-experiencia-de-usuario.md#35-explicabilidad--por-qué-esta-apuesta)).

---

## 7.6 Lo que este documento promete y lo que no

**Promete:** una arquitectura capaz de estimar probabilidades bien calibradas,
medirlas honestamente contra el mercado, publicar solo las discrepancias
estadísticamente significativas y dimensionar el capital de forma que la ruina
sea improbable.

**No promete:** que exista una ventaja explotable en todos los mercados, ni que
sea estable en el tiempo, ni que las casas permitan explotarla indefinidamente.
La Fase 0 existe precisamente para responder a esa pregunta con datos antes de
gastar un euro en producto.

Un sistema honesto en este dominio se reconoce por tres señales: **muestra sus
intervalos de confianza**, **puede decir que hoy no hay nada**, y **mide su
propio rendimiento con CLV en lugar de con el ROI del último mes**. Los tres
son requisitos bloqueantes de esta especificación.
