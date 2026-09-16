# 01 — Fuentes de datos y APIs

El sistema se alimenta de **seis familias de datos**. Cada una tiene su propia
frecuencia, criticidad y estrategia de ingesta. La regla de oro: *ninguna
familia es opcional para el motor de precios, pero todas deben degradarse con
elegancia* (si falta el clima, el modelo reprecia sin él y marca la señal con
menor confianza).

---

## 1.1 Familia A — Cuotas y mercado (criticidad: máxima)

Es la única familia cuya caída deja el producto sin función. Se divide en dos
subclases con propósitos distintos:

### A1. Libros "sharp" — la referencia de verdad

Son la estimación de consenso contra la que se mide el valor. No se apuesta
contra ellos: se usan como estimador.

| Proveedor | Acceso | Frecuencia | Uso en el sistema |
|---|---|---|---|
| **Pinnacle** | API REST oficial (`/v1/odds` con parámetro `since` para *delta polling*, `/v1/fixtures`, `/v1/line`) | 1–5 s con deltas | Estimador primario de probabilidad justa. Su **límite máximo publicado** se usa como proxy de confianza del propio libro |
| **Betfair Exchange** | Exchange Stream API (push TCP, mensajes `MCM`/`OCM`) + Betting API-NG | Push, < 200 ms | Precio de mercado real con volumen matcheado. Back/lay medio ponderado por volumen = probabilidad justa sin margen de casa |
| **Smarkets / Matchbook** | REST + WebSocket | 1 s | Validación cruzada y liquidez adicional en nicho |
| **Circa / BetOnline (US)** | Vía agregador | 5 s | Referencia para deportes US en Fase 2 |

> **Decisión de arquitectura.** La probabilidad de referencia se construye como
> media ponderada por inverso de varianza entre Pinnacle (devigged por método de
> Shin) y Betfair (back/lay mid ponderado por volumen disponible). Betfair pesa
> más cuando el volumen matcheado supera un umbral por competición.

### A2. Libros "soft" — donde se ejecuta la apuesta

| Vía | Proveedores | Coste orientativo | Notas |
|---|---|---|---|
| **Agregadores comerciales** | OddsJam, OddsMatrix, BetsAPI, The Odds API, Sportradar Odds (Betradar) | 200 €/mes (The Odds API) → 3.000+ €/mes (nivel *low-latency* de OddsJam) | The Odds API es suficiente para el MVP; para producción se necesita un feed *push* de baja latencia |
| **Feeds de afiliado** | Acuerdos directos con operador | Variable | Mejor latencia y datos de límites; exige contrato comercial |

**Requisitos funcionales de ingesta de cuotas:**

- `RF-A-01` Capturar **toda la serie temporal**, no solo el último precio. Cada
  cambio de precio es una fila inmutable con `(evento, mercado, selección, casa,
  cuota, línea, timestamp_proveedor, timestamp_ingesta)`.
- `RF-A-02` Registrar explícitamente la **cuota de apertura** y la **cuota de
  cierre** (último precio antes del inicio). Sin cierre no hay CLV y sin CLV no
  hay validación de estrategia.
- `RF-A-03` Distinguir *suspendido* de *sin datos*. Un mercado suspendido es
  información (suele preceder a una noticia).
- `RF-A-04` Normalizar líneas asiáticas a notación canónica (`-0.25`, `+1.75`)
  y mercados de totales por línea exacta, nunca agregando líneas distintas.
- `RF-A-05` Estimar la **latencia del proveedor** midiendo el desfase entre
  `timestamp_proveedor` y `timestamp_ingesta`; excluir de las señales de
  velocidad a los proveedores con p95 > 10 s.

---

## 1.2 Familia B — Datos de evento y estadísticas avanzadas

El nivel de granularidad determina el techo de calidad del modelo.

### B1. Nivel 1 — Datos de evento con coordenadas (preferente)

| Proveedor | Qué aporta | Coste orientativo |
|---|---|---|
| **StatsBomb** | Eventos con *freeze frames* (posición de todos los jugadores en el momento del disparo), xG propietario, OBV (*On-Ball Value*), presión, tipos de pase | 15.000–60.000 €/año |
| **Opta / Stats Perform** | El estándar de la industria. Eventos F24, xG, xA, secuencias, datos de 1.000+ competiciones | 20.000–100.000 €/año |
| **Wyscout (Hudl)** | Cobertura extraordinaria de ligas menores — clave para el edge en nicho | 5.000–20.000 €/año |
| **SkillCorner** | Datos físicos y de posicionamiento derivados de la retransmisión (velocidad, distancia, líneas defensivas) sin instalación en estadio | 10.000–40.000 €/año |

### B2. Nivel 2 — Fuentes abiertas / de bajo coste (viable para MVP y backtest)

| Fuente | Qué aporta | Licencia |
|---|---|---|
| **football-data.co.uk** | Histórico de resultados + **cuotas de cierre de múltiples casas desde 1993**. Imprescindible para el backtest inicial | Uso libre citando fuente |
| **Understat** | xG por partido y por disparo, 6 ligas | Scraping — revisar ToS |
| **FBref / StatsBomb open data** | xG, xA, métricas de posesión y progresión | Gratuito, atribución |
| **API-Football (api-sports.io)** | Fixtures, alineaciones, eventos, estadísticas básicas, amplia cobertura | 30–150 €/mes |
| **Transfermarkt** | Valor de plantilla, lesiones, sanciones | Scraping — revisar ToS |

> **Decisión.** El MVP se entrena con Nivel 2 (coste ~200 €/mes) y se valida la
> hipótesis de edge antes de comprometer un contrato de Nivel 1. El diseño del
> *feature store* abstrae el proveedor tras una interfaz común
> (`ShotEventProvider`, `MatchEventProvider`) para que el cambio sea una
> sustitución de adaptador, no una reescritura.

### B3. Métricas derivadas que el sistema debe calcular

No se consumen crudas: se calculan y versionan como features.

**Ataque / creación**
- `npxG` y `npxG/disparo` (excluyendo penaltis, que distorsionan medias).
- `xGOT` (*expected goals on target*, post-disparo) — separa calidad de
  definición de calidad de ocasión.
- `xT` (*expected threat*) y valor de progresión por carrera y por pase.
- xG de balón parado vs jugada abierta (procesos con persistencia muy distinta).

**Defensa / portería**
- `PSxG − GA` del portero: la métrica más útil y más malinterpretada. Tiene
  fuerte reversión a la media; el modelo debe aplicar *shrinkage* bayesiano.
- Altura de línea defensiva, PPDA, xG concedido por zona.

**Estabilidad**
- Media móvil exponencial de npxG a favor y en contra con **semivida
  configurable** (por defecto 10 partidos), separada por local/visitante.
- Ajuste por calidad de rival (*opponent-adjusted*), obligatorio: 8 partidos
  contra rivales de descenso no equivalen a 8 contra el top-4.

> ⚠️ **Advertencia metodológica.** Los goles son una medida ruidosa del
> rendimiento; xG es mejor pero también es un modelo con su propio error. El
> sistema nunca trata xG como verdad: lo propaga con su varianza asociada.

---

## 1.3 Familia C — Disponibilidad y contexto de plantilla

Es la familia con **mayor ratio señal/esfuerzo** del sistema y la que habilita
la ventaja por velocidad.

| Dato | Fuente | Ventana crítica |
|---|---|---|
| **Alineaciones confirmadas** | API-Football, Sportradar, feeds oficiales de club, cuentas X/Twitter de periodistas tier-1 | T−60 min. **El mayor movimiento de línea pre-partido del día** |
| **Alineaciones probables** | Rotowire, Sportsgambler, FotMob | T−48 h a T−2 h |
| **Lesiones y sanciones** | Transfermarkt, PhysioRoom, Premier Injuries, partes médicos oficiales | Continuo |
| **Carga de minutos y congestión** | Derivado interno de fixtures | Continuo |
| **Distancia y huso horario de viaje** | Cálculo geodésico desde coordenadas de estadio | Estático por fixture |
| **Días de descanso** | Derivado del calendario | Estático por fixture |

**Requisitos:**

- `RF-C-01` Modelo de **impacto por jugador ausente**: no basta con "falta
  Mbappé". Se cuantifica vía un rating de aportación (OBV/xT por 90 ajustado)
  y el sistema estima el delta de fuerza del equipo con el sustituto esperado.
- `RF-C-02` Pipeline de **NLP de noticias** sobre una lista curada de fuentes
  fiables (cuentas verificadas de periodistas, RSS oficiales de clubes) con
  clasificación `{lesión, duda, descarte, recuperación, rotación}` y extracción
  de entidad jugador. **Requiere confirmación por segunda fuente antes de
  disparar un reprecio automático**, para evitar manipulación por rumor.
- `RF-C-03` Toda noticia se marca con `first_seen_at`; es la base del análisis
  de "¿llegamos antes que el mercado?".

---

## 1.4 Familia D — Árbitros

Marginal para el 1X2, **determinante** para mercados de tarjetas y faltas, que
son precisamente los mercados de nicho con más margen explotable.

| Dato | Uso |
|---|---|
| Tarjetas amarillas/rojas por 90, ajustadas por competición | Predictor primario del mercado de tarjetas |
| Faltas señaladas por 90 y ratio faltas/tarjeta | Estilo de arbitraje |
| Penaltis concedidos por 90 | Cola de la distribución de goles |
| Sesgo local (diferencial de tarjetas local−visitante) | Ajuste de ventaja de campo |
| Uso e intervención de VAR | Cambia distribuciones históricas post-2019 |
| Histórico árbitro × equipo | **Usar con extrema cautela**: `n` suele ser < 10, es ruido casi puro. Exige *shrinkage* agresivo hacia la media de la competición |

Fuentes: WhoScored, Transfermarkt, FBref, federaciones nacionales, Sportradar.

- `RF-D-01` La designación arbitral se publica típicamente entre 72 h y 24 h
  antes. El sistema debe reprecia los mercados de tarjetas al recibirla.
- `RF-D-02` Todo agregado arbitral se calcula con *shrinkage* James-Stein hacia
  la media de la competición-temporada, con peso proporcional a `n` partidos.

---

## 1.5 Familia E — Meteorología y estadio

| Variable | Impacto documentado | Mercados afectados |
|---|---|---|
| **Viento (velocidad y ráfagas)** | El factor meteorológico de mayor efecto. Reduce precisión de pase largo y saques, aumenta córners | Totales, córners, NFL totals (Fase 2) |
| **Lluvia / estado del terreno** | Aumenta errores, reduce ritmo de juego | Totales, BTTS |
| **Temperatura extrema** | Reduce distancia recorrida e intensidad | Totales, córners |
| **Altitud** | La Paz, Quito, Bogotá, Denver: efecto sustancial y persistente | Todos |
| **Superficie (césped natural / artificial)** | Cambia ritmo y lesiones | Totales |

**Proveedores:** Visual Crossing (excelente histórico horario, ~35 €/mes),
Tomorrow.io, Meteomatics, OpenWeatherMap One Call.

- `RF-E-01` Consulta por **coordenadas exactas del estadio** y hora de saque
  local, no por ciudad.
- `RF-E-02` Almacenar tanto el **pronóstico en el momento `t`** como la
  **observación real** posterior. El backtest debe usar el pronóstico disponible
  en `t` (point-in-time), nunca la observación — error de *lookahead* clásico.
- `RF-E-03` Detectar y marcar estadios con cubierta cerrada o techo retráctil
  para anular el efecto meteorológico.

---

## 1.6 Familia F — Señales de mercado y flujo de dinero

Datos sobre el propio mercado, no sobre el partido. Alimentan el detector de
trampas.

| Señal | Fuente | Interpretación |
|---|---|---|
| Volumen matcheado en exchange | Betfair Stream API | Confianza en el precio; liquidez real ejecutable |
| *Steam move* (movimiento simultáneo en múltiples libros) | Derivado de la serie de cuotas | Dinero informado entrando |
| Divergencia % tickets vs % dinero | Action Network, Sports Insights (US) | Señal de *sharp* vs público |
| Evolución del límite máximo de Pinnacle | Pinnacle API | Su confianza sube cerca del evento |
| Dispersión entre casas | Derivado | Alta dispersión = incertidumbre genuina o error de alguien |

- `RF-F-01` Detección de `steam move`: ≥ 3 libros independientes moviendo en la
  misma dirección ≥ 2 % de probabilidad implícita en < 120 s.
- `RF-F-02` Detección de **línea obsoleta**: una casa que no ha reprecido ≥ 90 s
  después de un steam move confirmado. Es la señal de valor más limpia... y la
  más propensa a anulación por la casa.

---

## 1.7 El problema real: resolución de entidades

Es, en la práctica, la fuente número uno de bugs y de pérdida de dinero en este
tipo de sistemas. Cinco proveedores llaman al mismo club *Manchester United*,
*Man Utd*, *Manchester Utd FC*, *MUN* y `id=33`. Un cruce mal resuelto asigna
las estadísticas del rival al equipo, y el modelo produce con total confianza una
probabilidad invertida.

**Diseño del `entity-resolver`:**

1. **IDs canónicos internos** (`team_id`, `player_id`, `competition_id`,
   `venue_id`, `referee_id`) como única verdad. Ningún servicio aguas abajo
   toca un ID de proveedor.
2. **Tabla de mapeo** `provider_entity_map (provider, provider_id, entity_type,
   canonical_id, confidence, verified_by, verified_at)`.
3. **Emparejamiento automático** en cascada: (a) coincidencia exacta
   normalizada, (b) distancia de Jaro-Winkler sobre nombre normalizado + alias,
   (c) desambiguación por contexto (competición + temporada + fecha + rival).
4. **Umbral y cola humana.** Confianza < 0,95 → cuarentena en cola de revisión.
   Un fixture con alguna entidad sin resolver **no se pone a precio**; se marca
   `BLOCKED_ENTITY_RESOLUTION` y se alerta al operador.
5. **Test de regresión anti-inversión:** para cada partido resuelto se verifica
   que el resultado histórico conocido concuerda entre al menos dos proveedores.
   Una discrepancia bloquea el fixture.

---

## 1.8 Contratos de datos y control de calidad

Cada conector implementa la misma interfaz y publica métricas homogéneas.

```python
class DataSource(Protocol):
    source_id: str
    sla_freshness_seconds: int

    async def fetch(self, since: datetime) -> AsyncIterator[RawRecord]: ...
    def normalize(self, raw: RawRecord) -> CanonicalRecord: ...
    def validate(self, rec: CanonicalRecord) -> ValidationResult: ...
```

**Validaciones obligatorias (Great Expectations / Pandera):**

| Check | Regla | Acción al fallar |
|---|---|---|
| Suma de probabilidades implícitas | `1.00 ≤ Σ 1/cuota ≤ 1.25` | Descartar snapshot, alertar |
| Rango de cuota | `1.01 ≤ cuota ≤ 1000` | Descartar fila |
| Monotonía temporal | `timestamp` no decreciente por serie | Reordenar, marcar anomalía |
| Frescura | Edad del dato < SLA del proveedor | Degradar confianza del proveedor |
| Completitud de mercado | Todas las selecciones del mercado presentes | No poner a precio ese mercado |
| Coherencia xG | `0 ≤ xG_partido ≤ 8` | Cuarentena |
| Cobertura | ≥ 95 % de fixtures en alcance con datos completos | Alerta operativa |

**Niveles de almacenamiento (arquitectura medallón):**

- **Bronze** — payload crudo del proveedor, inmutable, particionado por
  `(source, fecha)` en S3/Parquet. Permite reprocesar el histórico cuando se
  corrige un bug de normalización, sin volver a pagar por los datos.
- **Silver** — registros canónicos, validados, con IDs internos resueltos.
- **Gold** — features listas para el modelo, con versión y linaje.

**Coste total estimado de datos (v1.0, 12 competiciones):**

| Partida | Mensual |
|---|---|
| Agregador de cuotas (nivel producción) | 400–900 € |
| API de fixtures/alineaciones/estadísticas | 150 € |
| Meteorología | 50 € |
| Histórico de backtest (pago único ~1.500 €) | — |
| **Total operativo** | **≈ 600–1.100 €/mes** |

Con proveedor de eventos Nivel 1 (Opta/StatsBomb) el coste sube a
**2.500–8.000 €/mes**, decisión que se pospone hasta validar el edge (ver
[§07.1](07-plan-de-entrega-kpis-y-cumplimiento.md#71-fases-de-entrega)).
