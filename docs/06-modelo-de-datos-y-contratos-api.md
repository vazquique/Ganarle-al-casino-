# 06 — Modelo de datos y contratos de API

## 6.1 Esquema relacional núcleo (PostgreSQL 16 + TimescaleDB)

```sql
-- ═══════════════════════════════ ENTIDADES CANÓNICAS ═════════════════════════

CREATE TABLE competition (
    competition_id   BIGSERIAL PRIMARY KEY,
    code             TEXT UNIQUE NOT NULL,        -- 'ESP_1', 'ENG_2', 'NOR_1'
    name             TEXT NOT NULL,
    country          CHAR(3) NOT NULL,            -- ISO 3166-1 alpha-3
    tier             SMALLINT NOT NULL,
    sport            TEXT NOT NULL DEFAULT 'football'
);

CREATE TABLE team (
    team_id          BIGSERIAL PRIMARY KEY,
    canonical_name   TEXT NOT NULL,
    short_name       TEXT,
    country          CHAR(3),
    aliases          TEXT[] NOT NULL DEFAULT '{}',
    founded_year     SMALLINT
);
CREATE INDEX ON team USING GIN (aliases);

CREATE TABLE venue (
    venue_id         BIGSERIAL PRIMARY KEY,
    name             TEXT NOT NULL,
    latitude         NUMERIC(9,6) NOT NULL,       -- necesario para clima exacto
    longitude        NUMERIC(9,6) NOT NULL,
    altitude_m       INTEGER,
    surface          TEXT CHECK (surface IN ('grass','artificial','hybrid')),
    has_roof         BOOLEAN NOT NULL DEFAULT FALSE,
    capacity         INTEGER
);

CREATE TABLE referee (
    referee_id       BIGSERIAL PRIMARY KEY,
    canonical_name   TEXT NOT NULL,
    country          CHAR(3)
);

-- Mapeo proveedor → canónico. La tabla más crítica del sistema (§01.7).
CREATE TABLE provider_entity_map (
    provider         TEXT     NOT NULL,
    provider_id      TEXT     NOT NULL,
    entity_type      TEXT     NOT NULL CHECK (entity_type IN
                          ('team','player','competition','venue','referee','fixture')),
    canonical_id     BIGINT   NOT NULL,
    confidence       REAL     NOT NULL CHECK (confidence BETWEEN 0 AND 1),
    verified_by      TEXT,                        -- NULL = automático
    verified_at      TIMESTAMPTZ,
    PRIMARY KEY (provider, provider_id, entity_type)
);
CREATE INDEX ON provider_entity_map (entity_type, canonical_id);

-- ═══════════════════════════════════ FIXTURES ════════════════════════════════

CREATE TABLE fixture (
    fixture_id       BIGSERIAL PRIMARY KEY,
    competition_id   BIGINT NOT NULL REFERENCES competition,
    season           TEXT   NOT NULL,
    kickoff_utc      TIMESTAMPTZ NOT NULL,
    home_team_id     BIGINT NOT NULL REFERENCES team,
    away_team_id     BIGINT NOT NULL REFERENCES team,
    venue_id         BIGINT REFERENCES venue,
    referee_id       BIGINT REFERENCES referee,
    status           TEXT NOT NULL DEFAULT 'scheduled'
                     CHECK (status IN ('scheduled','live','finished','postponed','cancelled')),
    -- bloqueos de integridad: un fixture bloqueado nunca se pone a precio
    block_reason     TEXT,
    home_goals       SMALLINT,
    away_goals       SMALLINT,
    home_xg          NUMERIC(5,3),
    away_xg          NUMERIC(5,3),
    CONSTRAINT distinct_teams CHECK (home_team_id <> away_team_id)
);
CREATE INDEX ON fixture (kickoff_utc) WHERE status = 'scheduled';
CREATE INDEX ON fixture (competition_id, season);

-- ════════════════════════════ SERIE TEMPORAL DE CUOTAS ═══════════════════════

CREATE TABLE odds_tick (
    fixture_id       BIGINT      NOT NULL,
    bookmaker        TEXT        NOT NULL,
    market           TEXT        NOT NULL,        -- '1X2','OU_GOALS','AH','BTTS','CORNERS','CARDS'
    line             NUMERIC(5,2),                -- 2.5, -0.25 … NULL en 1X2
    selection        TEXT        NOT NULL,        -- 'HOME','DRAW','AWAY','OVER','UNDER'
    odds             NUMERIC(8,3) NOT NULL CHECK (odds BETWEEN 1.01 AND 1000),
    max_stake        NUMERIC(10,2),               -- límite publicado (Pinnacle)
    matched_volume   NUMERIC(14,2),               -- volumen casado (exchange)
    is_suspended     BOOLEAN     NOT NULL DEFAULT FALSE,
    provider_ts      TIMESTAMPTZ NOT NULL,        -- reloj del proveedor
    ingested_ts      TIMESTAMPTZ NOT NULL DEFAULT now()
);
SELECT create_hypertable('odds_tick', 'provider_ts', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX ON odds_tick (fixture_id, market, selection, provider_ts DESC);
ALTER TABLE odds_tick SET (timescaledb.compress,
    timescaledb.compress_segmentby = 'fixture_id, bookmaker, market, selection');
SELECT add_compression_policy('odds_tick', INTERVAL '7 days');

-- Apertura y cierre materializados: sin ellos no hay CLV.
CREATE TABLE odds_summary (
    fixture_id       BIGINT NOT NULL,
    bookmaker        TEXT   NOT NULL,
    market           TEXT   NOT NULL,
    line             NUMERIC(5,2),
    selection        TEXT   NOT NULL,
    opening_odds     NUMERIC(8,3),
    opening_ts       TIMESTAMPTZ,
    closing_odds     NUMERIC(8,3),
    closing_ts       TIMESTAMPTZ,
    fair_closing_prob NUMERIC(6,5),               -- devigged, referencia de CLV
    PRIMARY KEY (fixture_id, bookmaker, market, line, selection)
);

-- ═══════════════════════════ PRECIOS DEL MODELO ══════════════════════════════

CREATE TABLE model_price (
    price_id         BIGSERIAL PRIMARY KEY,
    fixture_id       BIGINT NOT NULL REFERENCES fixture,
    market           TEXT   NOT NULL,
    line             NUMERIC(5,2),
    selection        TEXT   NOT NULL,
    prob             NUMERIC(6,5) NOT NULL CHECK (prob BETWEEN 0 AND 1),
    prob_lo          NUMERIC(6,5) NOT NULL,       -- percentil 5 de la posterior
    prob_hi          NUMERIC(6,5) NOT NULL,       -- percentil 95
    sigma            NUMERIC(6,5) NOT NULL,       -- incertidumbre total propagada
    market_prob      NUMERIC(6,5),                -- referencia sharp devigged
    blend_weight_mkt NUMERIC(4,3),                -- w_m aplicado
    model_version    TEXT   NOT NULL,
    pricing_run_id   UUID   NOT NULL,
    sim_seed         BIGINT NOT NULL,             -- reproducibilidad exacta
    feature_vector   JSONB  NOT NULL,             -- auditoría a 6 meses vista
    computed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ci_ordered CHECK (prob_lo <= prob AND prob <= prob_hi)
);
CREATE INDEX ON model_price (fixture_id, market, selection, computed_at DESC);

-- ═════════════════════════════ SEÑALES DE VALOR ══════════════════════════════

CREATE TABLE value_signal (
    signal_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    price_id         BIGINT NOT NULL REFERENCES model_price,
    fixture_id       BIGINT NOT NULL REFERENCES fixture,
    bookmaker        TEXT   NOT NULL,
    market           TEXT   NOT NULL,
    line             NUMERIC(5,2),
    selection        TEXT   NOT NULL,
    offered_odds     NUMERIC(8,3) NOT NULL,
    fair_odds        NUMERIC(8,3) NOT NULL,
    edge_pct         NUMERIC(6,3) NOT NULL,
    z_score          NUMERIC(6,3) NOT NULL,
    tier             CHAR(1) NOT NULL CHECK (tier IN ('A','B','C')),
    kelly_fraction   NUMERIC(6,5) NOT NULL,
    est_max_stake    NUMERIC(10,2),
    shap_top         JSONB,                       -- explicabilidad mostrada
    filters_passed   TEXT[] NOT NULL,
    published_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    expired_at       TIMESTAMPTZ,
    expiry_reason    TEXT                         -- 'ODDS_MOVED','KICKOFF','SUPPRESSED'
);
CREATE INDEX ON value_signal (published_at DESC) WHERE expired_at IS NULL;
CREATE INDEX ON value_signal (fixture_id);

-- ═══════════════════════ USUARIO, BANKROLL Y APUESTAS ════════════════════════

CREATE TABLE app_user (
    user_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_hash       TEXT NOT NULL UNIQUE,        -- email cifrado aparte
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    jurisdiction     CHAR(3) NOT NULL,
    age_verified_at  TIMESTAMPTZ,
    paper_mode       BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE bankroll_config (
    user_id          UUID PRIMARY KEY REFERENCES app_user ON DELETE CASCADE,
    bankroll         NUMERIC(12,2) NOT NULL,
    currency         CHAR(3) NOT NULL DEFAULT 'EUR',
    kelly_fraction   NUMERIC(4,3) NOT NULL DEFAULT 0.25
                     CHECK (kelly_fraction BETWEEN 0.05 AND 0.50),
    max_per_bet_pct     NUMERIC(5,3) NOT NULL DEFAULT 2.0,
    max_per_fixture_pct NUMERIC(5,3) NOT NULL DEFAULT 4.0,
    max_per_day_pct     NUMERIC(5,3) NOT NULL DEFAULT 8.0,
    max_open_pct        NUMERIC(5,3) NOT NULL DEFAULT 20.0,
    stop_loss_day_pct   NUMERIC(5,3),
    stop_loss_week_pct  NUMERIC(5,3),
    -- endurecer es inmediato; relajar espera 24 h (§04.6.2)
    pending_relax    JSONB,
    pending_relax_at TIMESTAMPTZ,
    cooldown_until   TIMESTAMPTZ,
    self_excluded_until TIMESTAMPTZ
);

CREATE TABLE bet (
    bet_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID   NOT NULL REFERENCES app_user ON DELETE CASCADE,
    signal_id        UUID   REFERENCES value_signal,   -- NULL = apuesta fuera del sistema
    fixture_id       BIGINT NOT NULL REFERENCES fixture,
    bookmaker        TEXT   NOT NULL,
    market           TEXT   NOT NULL,
    line             NUMERIC(5,2),
    selection        TEXT   NOT NULL,
    odds_taken       NUMERIC(8,3) NOT NULL,
    stake            NUMERIC(10,2) NOT NULL CHECK (stake > 0),
    suggested_stake  NUMERIC(10,2),                    -- para detectar sobre-stake
    is_paper         BOOLEAN NOT NULL DEFAULT FALSE,
    placed_at        TIMESTAMPTZ NOT NULL,
    settled_at       TIMESTAMPTZ,
    result           TEXT CHECK (result IN ('won','lost','void','half_won','half_lost','cashout')),
    pnl              NUMERIC(12,2),
    clv_pct          NUMERIC(6,3),                     -- calculado al cierre
    closing_fair_odds NUMERIC(8,3)
);
CREATE INDEX ON bet (user_id, placed_at DESC);
CREATE INDEX ON bet (fixture_id) WHERE settled_at IS NULL;
```

**Notas de diseño:**

- `odds_tick` es **append-only**. Nunca se actualiza una cuota: se inserta un
  nuevo tick. Es la única forma de reconstruir el estado del mercado en `t`.
- `model_price.feature_vector` almacena el vector completo usado. Encarece el
  almacenamiento y es innegociable: sin él, una señal de hace seis meses es
  inauditable y el análisis de fallos es imposible.
- `bet.signal_id` nullable es deliberado: registrar las apuestas que el usuario
  hace *fuera* del sistema es lo que permite el diagnóstico de
  [§04.6.1](04-gestion-de-riesgo-y-bankroll.md#461-detección-de-comportamiento-de-riesgo-tilt).

---

## 6.2 Contratos de eventos (Kafka / Redpanda)

Esquemas en **Avro** con Schema Registry y compatibilidad `BACKWARD`. Todos los
topics llevan sufijo de versión.

```json
// topic: canon.odds.v1   — clave: {fixture_id}:{bookmaker}:{market}:{selection}
{
  "type": "record", "name": "OddsTick", "namespace": "gac.canon.v1",
  "fields": [
    {"name": "fixture_id",     "type": "long"},
    {"name": "bookmaker",      "type": "string"},
    {"name": "market",         "type": "string"},
    {"name": "line",           "type": ["null", "double"], "default": null},
    {"name": "selection",      "type": "string"},
    {"name": "odds",           "type": "double"},
    {"name": "max_stake",      "type": ["null", "double"], "default": null},
    {"name": "matched_volume", "type": ["null", "double"], "default": null},
    {"name": "is_suspended",   "type": "boolean", "default": false},
    {"name": "provider_ts",    "type": {"type": "long", "logicalType": "timestamp-micros"}},
    {"name": "ingested_ts",    "type": {"type": "long", "logicalType": "timestamp-micros"}},
    {"name": "trace_id",       "type": "string"}
  ]
}
```

| Topic | Clave | Retención | Productor → consumidor |
|---|---|---|---|
| `raw.*.v1` | proveedor+id | 7 d | ingestores → normalizador |
| `canon.odds.v1` | fixture:book:market:sel | 30 d | normalizador → detector, timescale-sink |
| `canon.lineups.v1` | fixture | 30 d | lineup-ingestor → pricing-engine |
| `canon.weather.v1` | fixture | 30 d | weather-ingestor → pricing-engine |
| `prices.v1` | fixture:market:sel | 30 d | pricing-engine → detector |
| `signals.v1` | signal_id | 90 d | detector → alert, api, tracker |
| `signals.expired.v1` | signal_id | 90 d | detector → alert, api |
| `dlq.*` | — | 90 d | cualquiera → operaciones |

`RF-EV-01` — Todos los consumidores son **idempotentes** y toleran reentrega;
la clave de deduplicación es `(topic, key, provider_ts)`.
`RF-EV-02` — Cada mensaje propaga `trace_id` para trazabilidad extremo a extremo.

---

## 6.3 API pública

GraphQL para el cliente (el board necesita consultas con forma variable), REST
para integraciones y webhooks.

### 6.3.1 GraphQL — esquema núcleo

```graphql
type Query {
  valueSignals(filter: SignalFilter, first: Int = 50, after: String): SignalConnection!
  fixture(id: ID!): Fixture
  fixtures(competition: ID, from: DateTime, to: DateTime): [Fixture!]!
  myBets(filter: BetFilter, first: Int = 50): BetConnection!
  performance(segment: SegmentInput, from: DateTime, to: DateTime): PerformanceReport!
  bankroll: BankrollState!
}

type Mutation {
  recordBet(input: RecordBetInput!): Bet!
  settleBet(betId: ID!, result: BetResult!): Bet!
  updateBankrollConfig(input: BankrollConfigInput!): BankrollConfigResult!
  createAlertRule(input: AlertRuleInput!): AlertRule!
  startCooldown(days: Int!): BankrollState!
}

type Subscription {
  signalStream(filter: SignalFilter): SignalEvent!   # PUBLISHED | UPDATED | EXPIRED
  oddsStream(fixtureId: ID!, market: String!): OddsTick!
}

type ValueSignal {
  id: ID!
  fixture: Fixture!
  market: String!
  line: Float
  selection: String!
  bookmaker: String!
  offeredOdds: Float!
  fairOdds: Float!
  edgePct: Float!
  zScore: Float!
  tier: ConfidenceTier!                 # A | B | C
  probability: ProbabilityEstimate!     # value + lo + hi  (nunca un escalar suelto)
  suggestedStake: StakeRecommendation!
  explanation: Explanation!
  oddsHistory(window: Duration): [OddsPoint!]!
  publishedAt: DateTime!
  expiresHint: DateTime
}

type ProbabilityEstimate { value: Float!, lo: Float!, hi: Float!, sigma: Float! }

type StakeRecommendation {
  amount: Float!
  pctOfBankroll: Float!
  kellyFraction: Float!
  cappedBy: StakeCap          # PER_BET | PER_FIXTURE | PER_DAY | BOOK_LIMIT | CORRELATION | NONE
  correlatedWith: [ID!]!      # otras señales cuyo stake se ajustó conjuntamente
}

type Explanation {
  summary: String!            # frase generada desde SHAP, siempre trazable
  factors: [ShapFactor!]!
  marketWeight: Float!
}
```

`RF-API-01` — **`ProbabilityEstimate` nunca se serializa como escalar.** El tipo
obliga a transportar el intervalo; es una restricción de esquema, no una
convención, precisamente para que ningún cliente pueda mostrar un punto sin su
incertidumbre.

### 6.3.2 REST — endpoints de integración

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/v1/signals?tier=A&edge_min=3&market=1X2` | Señales activas (paginado por cursor) |
| `GET` | `/v1/signals/{id}` | Detalle con explicación y trayectoria de cuota |
| `GET` | `/v1/fixtures/{id}/prices` | Todos los precios del modelo del evento |
| `POST` | `/v1/bets` | Registro de apuesta |
| `PATCH` | `/v1/bets/{id}` | Liquidación / corrección |
| `GET` | `/v1/performance?segment=competition,market` | Rendimiento segmentado con IC bootstrap |
| `POST` | `/v1/webhooks` | Alta de webhook (`signal.published`, `signal.expired`, `bet.settled`, `risk.alert`) |

- Autenticación: OAuth 2.0 (usuarios) y claves de API con ámbitos (integraciones).
- Rate limiting: 60 rpm por defecto, 600 rpm en plan avanzado, cabeceras
  `X-RateLimit-*`.
- Errores conformes a **RFC 9457** (*Problem Details*).
- Webhooks firmados con HMAC-SHA256 y reintentos con retroceso exponencial.

### 6.3.3 Versionado

- GraphQL evoluciona por adición; los campos se marcan `@deprecated` durante
  90 días antes de eliminarse.
- REST versiona por ruta (`/v1`, `/v2`) con solapamiento mínimo de 6 meses.

---

## 6.4 Definiciones del *feature store* (Feast)

```python
fixture_entity = Entity(name="fixture_id", join_keys=["fixture_id"])

team_form = FeatureView(
    name="team_form",
    entities=[team_entity],
    ttl=timedelta(days=90),
    schema=[
        Field(name="npxg_for_ewma_10",       dtype=Float32),
        Field(name="npxg_against_ewma_10",   dtype=Float32),
        Field(name="npxg_for_ewma_5_oppadj", dtype=Float32),
        Field(name="ppda",                   dtype=Float32),
        Field(name="field_tilt",             dtype=Float32),
        Field(name="setpiece_xg_share",      dtype=Float32),
        Field(name="gk_psxg_minus_ga_shrunk",dtype=Float32),
        Field(name="att_rating_posterior",   dtype=Float32),
        Field(name="def_rating_posterior",   dtype=Float32),
        Field(name="rating_posterior_sd",    dtype=Float32),   # alimenta el stake
    ],
    source=team_form_source,   # con timestamp de evento → point-in-time join
)
```

`RF-FS-01` — **Toda `FeatureView` declara su `event_timestamp`.** Una feature sin
marca temporal de evento no puede participar en un *point-in-time join* y queda
prohibida en el sistema: es la puerta de entrada del *lookahead bias*.
`RF-FS-02` — Test de CI que verifica, sobre una muestra de 1.000 fixtures, que
el vector materializado offline coincide bit a bit con el servido online para el
mismo instante. La divergencia train/serve es un fallo silencioso que degrada
el modelo sin producir ningún error.
