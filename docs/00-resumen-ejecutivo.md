# 00 — Resumen ejecutivo y alcance

## 0.1 Problema

El apostador que busca rentabilidad a largo plazo se enfrenta a tres barreras
que ninguna herramienta de consumo resuelve bien a la vez:

1. **Barrera informativa.** Los datos que predicen resultados (xG ajustado por
   calidad de disparo, alineaciones confirmadas, carga de minutos, perfil del
   árbitro, viento a nivel de estadio) están dispersos entre proveedores
   distintos, con identificadores incompatibles y latencias distintas.
2. **Barrera estadística.** Convertir esos datos en una probabilidad
   **calibrada** —no solo "acertada"— y compararla correctamente con un precio
   que lleva margen incorporado exige un aparato metodológico que el usuario
   medio no tiene.
3. **Barrera conductual.** Incluso con una ventaja real del 2 %, un mal
   dimensionamiento de apuesta o una racha negativa gestionada con *tilt*
   destruyen la bankroll antes de que la ventaja se materialice.

## 0.2 Propuesta de producto

Una plataforma web y móvil que ejecuta, de forma continua y automática:

```
Ingesta multi-proveedor → Normalización y resolución de entidades
   → Feature store con corrección point-in-time
   → Motor de precios (modelo generativo + ML + consenso de mercado)
   → Detector de valor con umbral dinámico por incertidumbre
   → Alertas, panel de decisión y control de bankroll
   → Registro de apuestas, CLV y diagnóstico de rendimiento
```

## 0.3 Hipótesis de valor y dónde está realmente el edge

Un documento técnico honesto debe declarar dónde es plausible ganar y dónde no.
Los mercados principales de las cinco grandes ligas europeas y de la NBA/NFL son
**extremadamente eficientes**: el margen de Pinnacle en un 1X2 de Premier League
ronda el 2,0–2,5 % y su línea de cierre es, empíricamente, uno de los mejores
predictores públicos disponibles. Batirla de forma sistemática es difícil.

El edge explotable se concentra, por orden de plausibilidad:

| Fuente de edge | Descripción | Dificultad |
|---|---|---|
| **Velocidad ante noticia** | Alineaciones confirmadas, lesión de última hora, cambio meteorológico. La ventana entre la noticia y el reajuste de los libros blandos es de segundos a minutos. | Media — es un problema de ingeniería de latencia |
| **Mercados de nicho** | Segundas divisiones, ligas escandinavas/sudamericanas, córners, tarjetas, tiros a puerta, props de jugador. Menos atención del *sharp money*, márgenes mayores pero líneas peores. | Media |
| **Arbitraje de precio soft vs sharp** | La cuota justa derivada de Pinnacle/Betfair frente a la mejor cuota de una casa blanda. Es la señal más robusta y reproducible. | Baja técnicamente, alta operativamente (limitación de cuentas) |
| **Modelización superior** | Un modelo propio que capture algo que el mercado infravalora (p. ej. calidad de portero sobre PSxG, efecto de viento en córners). | Alta |
| **Errores de tarificación** | Líneas obsoletas, *palps*, incoherencias entre mercados del mismo partido. | Baja, pero efímero y frecuentemente anulado por la casa |

**Riesgo operativo estructural:** las casas blandas limitan o cierran las cuentas
de los usuarios rentables. Una plataforma que ignora esto vende una fantasía. El
producto lo trata como requisito de primera clase: ver
[§04.7](04-gestion-de-riesgo-y-bankroll.md#47-riesgo-operativo-limitación-de-cuentas).

## 0.4 Alcance

### En alcance (v1.0)

- Fútbol: 12 competiciones (Big-5 + Eredivisie, Primeira Liga, Championship,
  Brasileirão, Liga MX, MLS, Champions League).
- Mercados: 1X2, Over/Under goles, hándicap asiático, BTTS, córners, tarjetas.
- Pre-partido desde T−72 h hasta el saque inicial.
- Panel web responsive + PWA con notificaciones push.
- Tracker de apuestas manual + importación CSV, CLV automático.
- Módulo completo de bankroll y juego responsable.

### Fuera de alcance (v1.0, planificado)

- **Live / in-play.** Exige tracking de balón a 25 Hz, latencias de decenas de
  milisegundos y modelos de estado de partido. Fase 3.
- **Colocación automática de apuestas.** Prohibida por los términos de servicio
  de prácticamente todas las casas blandas; solo viable vía API de exchange
  (Betfair). Se deja como integración opcional y explícita en Fase 3.
- **Tenis, baloncesto, NFL, eSports.** Fase 2–3.
- **Producto social / tipsters.** Explícitamente descartado: incentiva el
  seguimiento acrítico y entra en conflicto con la normativa publicitaria de
  juego.

## 0.5 Usuarios objetivo

| Perfil | Necesidad dominante | Funcionalidad crítica |
|---|---|---|
| **Analítico** (bankroll 2.000–50.000 €, 200+ apuestas/mes) | Volumen de señales filtradas y CLV | Value Board, API, alertas, exportación |
| **Recreativo informado** (bankroll < 2.000 €) | Contexto y disciplina | Ficha de partido, explicabilidad, límites duros |
| **Sindicato pequeño** (2–5 personas) | Coordinación y control de exposición | Multi-cuenta, exposición agregada, roles |

## 0.6 Requisitos no funcionales de cabecera

| Requisito | Objetivo |
|---|---|
| Latencia tick de cuota → alerta (p95) | < 800 ms |
| Latencia noticia de alineación → reprecio (p95) | < 5 s |
| Frescura de datos de cuotas | < 3 s para libros con stream, < 20 s con polling |
| Disponibilidad del panel | 99,5 % mensual |
| Cobertura de partidos con modelo calibrado | ≥ 95 % de los eventos en alcance |
| Reproducibilidad del backtest | Determinista bit a bit dado un `run_id` |
| RGPD | Datos de apuestas cifrados en reposo, derecho de supresión en < 30 días |

## 0.7 Principios de diseño

1. **La incertidumbre es un ciudadano de primera clase.** Ninguna probabilidad
   se transporta por el sistema sin su intervalo creíble asociado.
2. **Corrección point-in-time o nada.** Cualquier feature debe poder
   reconstruirse tal como era en el instante `t`. El *lookahead bias* es el
   fallo más caro y más silencioso de este dominio.
3. **Un solo modelo generativo por partido.** Todos los mercados se derivan de
   la misma simulación, garantizando coherencia (si el modelo dice O2.5 = 55 %,
   la suma de sus marcadores exactos lo respeta).
4. **El mercado es un competidor informado, no un adversario tonto.** Se usa
   como *prior* y como referencia de validación, no se ignora.
5. **El producto debe poder decir "hoy no hay nada".** Un panel vacío es un
   resultado válido y es señal de honestidad del sistema.
