# Ganarle al casino — Plataforma de Análisis de Mercados Deportivos

> **Documento de Requerimientos Técnicos (TRD) v1.0 — 16 de septiembre de 2026**

Especificación técnica de una plataforma de análisis cuantitativo de mercados
deportivos cuyo objetivo es **estimar probabilidades reales con la mejor
calibración posible** y **detectar discrepancias estadísticamente significativas
frente a las cuotas ofrecidas** por las casas de apuestas (*value bets*),
acompañadas de una gestión de capital disciplinada.

---

## Herramienta en vivo

**[▶ Libro de Momios](https://claude.ai/artifact/UwfxhLZBeULFiT9Duwe7Av)** — registro de picks que
lleva la cuenta de aciertos y fallos y calcula la ganancia o pérdida real según el momio de cada
pick, además de yield y CLV. Es la implementación mínima de los conceptos de los documentos 04
(seguimiento de resultados) y 02 (CLV como métrica de validación).

Código fuente: [`app/libro-de-momios.html`](app/libro-de-momios.html).

---

## Índice de la especificación

| # | Documento | Contenido |
|---|-----------|-----------|
| 00 | [Resumen ejecutivo y alcance](docs/00-resumen-ejecutivo.md) | Tesis del producto, hipótesis de valor, alcance MVP, límites honestos |
| 01 | [Fuentes de datos y APIs](docs/01-fuentes-de-datos-y-apis.md) | Proveedores, contratos de datos, ingesta, resolución de entidades |
| 02 | [Motor de análisis y algoritmos](docs/02-motor-de-analisis-y-algoritmos.md) | Devigging, modelos generativos, ML, blending, filtrado de ruido, validación |
| 03 | [Interfaz y experiencia de usuario](docs/03-interfaz-y-experiencia-de-usuario.md) | Value Board, alertas, gráficos de línea, mapas de calor, explicabilidad |
| 04 | [Gestión de riesgo y bankroll](docs/04-gestion-de-riesgo-y-bankroll.md) | Kelly fraccional, límites, stop-loss, detección de tilt, juego responsable |
| 05 | [Arquitectura técnica](docs/05-arquitectura-tecnica.md) | Servicios, stack, latencias, despliegue, observabilidad, seguridad |
| 06 | [Modelo de datos y contratos de API](docs/06-modelo-de-datos-y-contratos-api.md) | Esquema SQL, eventos Kafka, API pública, feature store |
| 07 | [Plan de entrega, KPIs y cumplimiento](docs/07-plan-de-entrega-kpis-y-cumplimiento.md) | Fases, criterios de aceptación, equipo, costes, legal |

---

## La tesis en una página

Una casa de apuestas no publica probabilidades: publica **precios con margen**
(*overround*). Si una casa ofrece 2,10 a un resultado, su probabilidad implícita
bruta es 47,6 %, pero una vez retirado el margen la probabilidad "justa" que el
mercado le asigna puede ser 45,0 %. Existe valor cuando **nuestra estimación de
la probabilidad real supera de forma estadísticamente significativa la
probabilidad justa implícita en el precio disponible**:

```
EV = p_modelo × (cuota − 1) − (1 − p_modelo)
Valor cuando:  EV > 0  Y  edge > k · σ(edge)
```

El error clásico —y la razón por la que la mayoría de sistemas de *value betting*
fracasan— es tratar `p_modelo` como si fuera exacta. Un modelo con un error de
calibración del 3 % genera cientos de "value bets" falsas al día en mercados
cuyo margen real es del 2 %. **Este producto se diseña alrededor de la
incertidumbre del estimador, no solo de su valor central.**

La arquitectura implementa tres capas de defensa contra el ruido:

1. **Referencia de mercado eficiente.** Pinnacle y Betfair Exchange se usan como
   estimador de consenso de baja varianza. El valor se mide contra su línea sin
   margen, no contra cero.
2. **Modelo generativo propio.** Simulación Monte Carlo a nivel de disparo
   (Dixon-Coles / bayesiano jerárquico sobre xG) que produce *todos* los mercados
   de forma internamente coherente.
3. **Mezcla calibrada y umbral dinámico.** Las dos señales se combinan con pesos
   aprendidos, y el umbral de publicación depende de la incertidumbre posterior
   de cada estimación.

La métrica primaria de éxito **no es el ROI a corto plazo** —demasiado ruidoso
para ser informativo con menos de 1.000 apuestas—, sino el **CLV (Closing Line
Value)**: la capacidad sistemática de conseguir un precio mejor que el de cierre
del mercado. Ver [§02.8](docs/02-motor-de-analisis-y-algoritmos.md#28-validación-y-backtesting).

---

## Aviso

Esta plataforma es una **herramienta de análisis estadístico**. No garantiza
beneficios, no opera como operador de juego, no acepta ni cursa apuestas y no
gestiona fondos de terceros. Las apuestas deportivas implican riesgo real de
pérdida de capital. El sistema incorpora por diseño controles de juego
responsable descritos en el [documento 04](docs/04-gestion-de-riesgo-y-bankroll.md)
y las obligaciones regulatorias en el
[documento 07](docs/07-plan-de-entrega-kpis-y-cumplimiento.md#75-marco-legal-y-cumplimiento).
