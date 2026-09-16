# 04 — Gestión de riesgo y bankroll

> Un modelo con ventaja real del 3 % puede arruinar a un usuario en seis semanas
> si el dimensionamiento de apuesta es incorrecto. **La gestión de capital no es
> una funcionalidad accesoria: es la mitad del producto.**

---

## 4.1 Dimensionamiento base: criterio de Kelly

El criterio de Kelly maximiza la tasa de crecimiento logarítmico esperado del
capital. Para una apuesta simple a cuota decimal `o` con probabilidad real `p`:

```
b  = o − 1                     (ganancia neta por unidad)
q  = 1 − p
f* = (p·b − q) / b  =  (p·o − 1) / (o − 1)  =  edge / (o − 1)
```

**Kelly completo es matemáticamente óptimo y prácticamente suicida.** Dos
razones:

1. **`p` no se conoce, se estima.** Kelly asume `p` exacta. Con un error de
   estimación del 2 % en una apuesta a cuota 3,00, el `f*` calculado puede
   duplicar el óptimo real — y sobreapostar es mucho más destructivo que
   infraapostar (la curva de crecimiento cae de forma abrupta pasado el óptimo y
   se vuelve negativa en `2·f*`).
2. **La volatilidad es intolerable.** Con Kelly completo, la probabilidad de
   sufrir en algún momento una caída del 50 % de la bankroll es del **50 %**.

### 4.1.1 Kelly fraccional

Se apuesta `f = c · f*`, con `c` la fracción de Kelly. Aproximación en tiempo
continuo (Thorp) para la probabilidad de sufrir **alguna vez** una caída hasta
la fracción `x` de la bankroll inicial:

```
P(drawdown hasta x) = x^(2/c − 1)
```

| Fracción `c` | P(caída al 50 %) | P(caída al 25 %) | % del crecimiento óptimo |
|---|---|---|---|
| 1,00 (completo) | 50,0 % | 25,0 % | 100 % |
| 0,50 (medio) | 12,5 % | 1,6 % | 75 % |
| **0,25 (cuarto)** | **0,8 %** | **0,006 %** | **44 %** |
| 0,125 (octavo) | 0,003 % | ~0 % | 23 % |

> **Decisión de producto: `c = 0,25` por defecto**, configurable en el rango
> `[0,10 – 0,50]`. Por encima de 0,50 el sistema **rechaza** la configuración y
> explica por qué. Sacrificar el 56 % del crecimiento teórico a cambio de
> reducir el riesgo de caída al 50 % desde 1 de cada 2 hasta 1 de cada 128 es,
> para un usuario real con un modelo imperfecto, un intercambio obviamente
> favorable.

### 4.1.2 Kelly ajustado por incertidumbre — el ajuste que casi nadie hace

El motor no entrega un `p̂` puntual: entrega una **distribución posterior**
([§02.2.2](02-motor-de-analisis-y-algoritmos.md#222-producción-modelo-bayesiano-jerárquico-sobre-xg)).
El stake se calcula maximizando el crecimiento logarítmico esperado **sobre toda
la posterior**, no sobre su media:

```python
# f_opt maximiza E_posterior[ log(1 + f·(o−1)·1{gana} − f·1{pierde}) ]
def kelly_bayesiano(p_samples: np.ndarray, odds: float) -> float:
    b = odds - 1.0
    def growth(f):
        # esperanza sobre las muestras posteriores de p
        return -np.mean(p_samples * np.log1p(f * b) + (1 - p_samples) * np.log1p(-f))
    return minimize_scalar(growth, bounds=(0.0, 0.99 / 1.0), method="bounded").x
```

Efecto práctico: dos señales con el mismo edge del 5 % pero incertidumbres
distintas reciben stakes muy distintos. La señal de un equipo recién ascendido
con 6 partidos jugados (posterior ancha) recibe una fracción del stake de una
señal sobre un equipo con 200 partidos de historial. **Esto es exactamente la
distinción que un sistema basado en `edge` puntual es incapaz de hacer, y es la
principal causa de ruina de los apostadores con modelo propio.**

---

## 4.2 Límites duros (se aplican después de Kelly, siempre)

Kelly propone; los límites disponen. Todo stake pasa por esta cascada:

```python
stake = kelly_bayesiano(p_samples, odds) * fraccion_kelly * bankroll
stake = min(stake, bankroll * MAX_POR_APUESTA)          # 2,0 % por defecto
stake = min(stake, limite_estimado_casa * 0.80)         # no delatar el modelo
stake = min(stake, exposicion_restante_partido)         # 4,0 % por partido
stake = min(stake, exposicion_restante_dia)             # 8,0 % por día
stake = min(stake, exposicion_restante_correlacionada)  # ver §4.3
stake = 0 if tier == "C" else stake                     # tier C nunca automático
stake = redondear_discreto(stake)                       # ver §4.7
if stake < STAKE_MINIMO: descartar_senal()
```

| Límite | Valor por defecto | Rango configurable |
|---|---|---|
| Máximo por apuesta | 2,0 % bankroll | 0,5 – 5,0 % |
| Máximo por partido (todos los mercados) | 4,0 % | 1,0 – 8,0 % |
| Máximo por día | 8,0 % | 2,0 – 20,0 % |
| Máximo por competición y semana | 15,0 % | 5,0 – 30,0 % |
| Exposición abierta simultánea | 20,0 % | 5,0 – 40,0 % |
| Nº máximo de apuestas abiertas | 25 | 5 – 100 |

`RF-BR-01` — Los límites se evalúan **sobre la exposición abierta real**, no
sobre el histórico. `RF-BR-02` — La bankroll se recalcula tras cada liquidación;
Kelly es proporcional al capital actual, lo que produce de forma natural la
reducción de stakes en rachas negativas (propiedad deseable y automática).

---

## 4.3 Kelly con apuestas correlacionadas

Tres señales del mismo partido (Over 2.5, BTTS Sí, y victoria del local) **no
son independientes**: pueden perder las tres a la vez. Aplicar Kelly individual
a cada una sobreapuesta gravemente el partido.

La solución es elegante porque el insumo ya existe: las **50.000 simulaciones
Monte Carlo del partido** ([§02.2.3](02-motor-de-analisis-y-algoritmos.md#223-simulación-monte-carlo-a-nivel-de-disparo))
son exactamente una muestra de la distribución conjunta de resultados. El
dimensionamiento conjunto se plantea como un problema de optimización convexa:

```
maximizar   (1/S) · Σ_s  log( 1 + Σ_i f_i · r_i(s) )
sujeto a    f_i ≥ 0,  Σ_i f_i ≤ F_max,  f_i ≤ cap_i

donde  r_i(s) = (o_i − 1)  si la apuesta i gana en el escenario s
               −1           en caso contrario
```

Es una maximización cóncava (log de una función afín) → programa convexo
resoluble con CVXPY/ECOS en milisegundos para `n ≤ 30` apuestas.

`RF-BR-03` — La optimización conjunta se ejecuta **por partido** y, una vez al
día, **por cartera completa** considerando correlaciones entre partidos
(mismo día, misma competición, mercados de goles correlacionados con la
tendencia general de la jornada).
`RF-BR-04` — La UI muestra el resultado como *"3 señales en Girona–Betis:
stake conjunto 58 € (individualmente sumarían 104 € — reducido por
correlación)"*, explicando el ajuste en lugar de aplicarlo en silencio.

---

## 4.4 Simulador de bankroll — antes de arriesgar, no después

`RF-BR-05` — Al configurar el staking, y de nuevo cada vez que el usuario lo
modifica, se ejecuta una simulación de 10.000 trayectorias con el edge histórico
real del usuario (o el del sistema si aún no tiene historial) y se muestra:

| Salida | Presentación |
|---|---|
| Distribución de bankroll a 6 y 12 meses | percentiles 5 / 25 / 50 / 75 / 95 |
| **Probabilidad de ruina** (caída bajo el 20 % del capital) | cifra protagonista |
| Drawdown máximo esperado (p95) | valor + duración esperada en apuestas |
| Tiempo de recuperación tras un drawdown p95 | en apuestas y en semanas |
| Probabilidad de estar en pérdidas a 6 meses **aun teniendo ventaja real** | cifra explícita |

La última fila es deliberadamente incómoda y deliberadamente prominente. Con un
yield real del 3 % y 200 apuestas, la probabilidad de terminar en negativo supera
el 33 %. **Un usuario que no interioriza esto antes de empezar abandonará o
entrará en tilt cuando ocurra.**

---

## 4.5 Distinguir la varianza del deterioro del modelo

El diagnóstico crítico —y el que ninguna herramienta de consumo ofrece— es
separar "estoy teniendo mala suerte" de "mi sistema ha dejado de funcionar".
Se resuelve con dos señales independientes:

| | **CLV positivo** | **CLV negativo** |
|---|---|---|
| **ROI positivo** | ✅ Sistema sano. Continuar. | ⚠️ Suerte. El ROI no es atribuible a ventaja; esperar regresión. |
| **ROI negativo** | 🟡 **Varianza.** La selección es correcta, el resultado aún no ha convergido. No cambiar nada. | 🔴 **Deterioro real.** Suspender staking automático e investigar. |

**Implementación:**

- `RF-BR-06` **CUSUM sobre el CLV** con ventana de 200 apuestas. Alerta cuando
  la suma acumulada de desviaciones negativas supera el umbral `h = 5σ`.
- `RF-BR-07` Alerta diferenciada por causa con el diagnóstico de la tabla, en
  texto explícito, nunca dejada a la interpretación del usuario.
- `RF-BR-08` Si el deterioro se confirma (CLV negativo sostenido), el staking
  automático se **suspende** y el sistema lo comunica: *"Hemos suspendido las
  recomendaciones de stake en el segmento LaLiga–córners. El CLV lleva 240
  apuestas en negativo, lo que indica un problema del modelo y no mala suerte."*

---

## 4.6 Controles de conducta y juego responsable

Esta sección no es un anexo de cumplimiento: es una **funcionalidad de
protección del capital del usuario y de la viabilidad del producto**.

### 4.6.1 Detección de comportamiento de riesgo (*tilt*)

Motor de reglas con features conductuales calculadas en tiempo real:

| Señal | Regla de detección | Intervención |
|---|---|---|
| **Persecución de pérdidas** | Stake medio de las últimas 5 apuestas > 1,8× el stake medio de 30 días, tras una pérdida neta en 24 h | Aviso modal con la comparativa; el registro exige confirmación adicional |
| **Aumento de frecuencia** | Nº de apuestas en 24 h > p95 histórico del usuario | Aviso suave + resumen de la sesión |
| **Apuestas fuera de sistema** | Apuestas registradas que no provienen de una señal publicada > 40 % en 7 días | Aviso: *"El 62 % de tus apuestas recientes no vienen del modelo. Tu CLV en esas apuestas es −1,8 %."* |
| **Apuestas nocturnas** | Actividad entre 01:00 y 06:00 local fuera del patrón habitual | Aviso + opción de activar ventana de silencio |
| **Sobre-stake** | Stake introducido > 2× el recomendado | **Fricción dura**: confirmación escrita del importe |
| **Recuperación acelerada** | Aumento de la fracción de Kelly configurada tras un drawdown > 15 % | Bloqueo de 24 h del cambio + explicación del efecto sobre la ruina |
| **Sesión prolongada** | > 90 min continuos en la app | Recordatorio de tiempo con resumen de P&L de la sesión |

`RF-BR-09` — **Todos los avisos son informativos y con datos concretos del
propio usuario, nunca moralizantes.** Un mensaje del tipo *"quizá deberías
parar"* se ignora; *"tus apuestas fuera del modelo llevan −340 € en 60 días"*
cambia comportamientos.

### 4.6.2 Límites autoimpuestos y periodos de enfriamiento

| Herramienta | Comportamiento |
|---|---|
| **Stop-loss diario / semanal / mensual** | Al alcanzarse, la app oculta el Value Board y muestra el resumen de rendimiento. Configurable por el usuario. |
| **Stop-win opcional** | Igual, en positivo (protege contra la devolución de ganancias). |
| **Asimetría de cambios** | Endurecer un límite es **inmediato**; relajarlo requiere **24 h de espera**. Esta asimetría es el mecanismo central: impide la decisión impulsiva en caliente. |
| **Enfriamiento** | Pausa autoimpuesta de 24 h / 7 / 30 días. Irrevocable durante su vigencia. |
| **Autoexclusión** | Cierre de cuenta de 6 meses o permanente, irreversible, con borrado de datos conforme a RGPD a petición. |
| **Recordatorio de realidad** | Resumen semanal automático con P&L real, nº de apuestas, tiempo en app y CLV. Enviado siempre, también (y sobre todo) en semanas negativas. |

`RF-BR-10` — El producto **no incluye**, en ningún caso: bonos, promociones de
depósito, funciones de "recupera tu racha", gamificación de la actividad de
apuesta (rachas, logros, niveles), ni tablas de clasificación de usuarios por
beneficio. Todos ellos son mecanismos documentados de incremento del juego
problemático y, en varias jurisdicciones, de riesgo regulatorio directo.

`RF-BR-11` — Verificación de edad (18+) en el registro y bloqueo por
geolocalización en jurisdicciones donde el producto no puede operar. Enlace
permanente y visible en el pie a recursos de ayuda
(FEJAR / Juego Seguro en España, GamCare / BeGambleAware en Reino Unido,
según la jurisdicción detectada).

---

## 4.7 Riesgo operativo: limitación de cuentas

La ventaja detectada es inútil si la casa no acepta el stake. Las casas blandas
identifican y limitan a los usuarios rentables, habitualmente en semanas. Un
producto serio lo trata como parte del modelo de riesgo, no como una sorpresa.

**Lo que el sistema hace:**

- `RF-BR-12` **Modelo de límite por casa y usuario.** Se estima el stake máximo
  aceptado por cada casa a partir del histórico de aceptaciones y rechazos del
  propio usuario, y las señales se dimensionan **al 80 % del límite estimado**,
  no al máximo. El stake sugerido nunca supera lo que la cuenta puede absorber.
- `RF-BR-13` **Redondeo discreto.** Un stake de `47,83 €` calculado por Kelly se
  presenta como `50 €`. Las cantidades con decimales exactos son un marcador
  evidente de staking algorítmico.
- `RF-BR-14` **Diversificación obligatoria.** El sistema reparte el volumen
  entre las casas disponibles del usuario y avisa cuando la concentración en una
  sola supera el 40 % del volumen mensual.
- `RF-BR-15` **Panel de salud de cuentas.** Seguimiento por casa de: stake medio
  aceptado, tasa de rechazo, evolución del límite. Una caída sostenida del
  límite se muestra como aviso con su lectura: *"Bet365 ha reducido tu límite
  efectivo un 60 % en 3 semanas. Prepara alternativas."*
- `RF-BR-16` **Ruta al exchange.** El producto expone de forma explícita que
  el destino sostenible del apostador rentable es el intercambio (Betfair,
  Smarkets, Matchbook), donde no existe limitación por rentabilidad, a cambio de
  comisión y de menor liquidez en nicho. Todos los cálculos de valor y de Kelly
  incorporan la comisión efectiva del exchange cuando esa es la vía elegida.

**Lo que el sistema no hace:** no automatiza la colocación de apuestas en casas
blandas, no gestiona cuentas de terceros ni facilita el uso de cuentas a nombre
de otras personas. Son violaciones de los términos de servicio de los operadores
y, según la jurisdicción, pueden constituir fraude. La única integración de
colocación automática prevista es la **API oficial de exchange**, que sí está
diseñada para uso programático.

---

## 4.8 Resumen de requisitos del módulo

| ID | Requisito | Prioridad |
|---|---|---|
| `RF-BR-01..02` | Cascada de límites sobre exposición real, bankroll dinámica | Must |
| `RF-BR-03..04` | Kelly conjunto sobre escenarios Monte Carlo, explicado en UI | Must |
| `RF-BR-05` | Simulador de ruina previo a la configuración | Must |
| `RF-BR-06..08` | Diagnóstico CLV/ROI con CUSUM y suspensión automática | Must |
| `RF-BR-09` | Detección de tilt con intervención basada en datos propios | Must |
| `RF-BR-10..11` | Ausencia de gamificación, límites asimétricos, autoexclusión, 18+ | Must (legal) |
| `RF-BR-12..16` | Gestión de límites de cuenta y ruta a exchange | Should |
