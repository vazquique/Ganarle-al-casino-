# 05 — Arquitectura técnica

## 5.1 Vista general

Arquitectura **orientada a eventos**, porque el dominio lo es: todo es un flujo
de cambios de precio y de estado del mundo. Un diseño petición-respuesta obliga
a *polling* en todas las capas y hace imposible el presupuesto de latencia.

```
┌─────────────── INGESTA ────────────────┐
│ odds-ingestor    (Go)  ── stream/poll  │──┐
│ stats-ingestor   (Py)  ── batch        │  │
│ lineup-ingestor  (Py)  ── stream+NLP   │  │   Kafka / Redpanda
│ weather-ingestor (Py)  ── cron         │  │   ┌──────────────────┐
│ referee-ingestor (Py)  ── cron         │  ├──►│ raw.odds.v1      │
└────────────────────────────────────────┘  │   │ raw.events.v1    │
                                            │   │ raw.lineups.v1   │
┌─────────── NORMALIZACIÓN ──────────────┐  │   │ raw.weather.v1   │
│ entity-resolver  (Py)                  │◄─┘   └────────┬─────────┘
│ validator        (Py, Pandera)         │               │
│ canonicalizer    (Py)                  │──────────────►│ canon.*.v1
└────────────────────────────────────────┘               │
                                                         ▼
┌──────────── ALMACENAMIENTO ──────────────────────────────────────────┐
│ PostgreSQL 16 + TimescaleDB   series de cuotas, entidades, apuestas  │
│ ClickHouse                    analítica OLAP, backtest, mapas calor  │
│ Redis                         features online, caché, rate limiting  │
│ S3 + Iceberg/Parquet          data lake bronze/silver/gold           │
│ Feast                         feature store (online Redis / offline) │
└──────────────────────────────────────────────────────────────────────┘
                                                         │
┌──────────────── ANÁLISIS ───────────────────────────────▼────────────┐
│ feature-builder   (Py, Dagster)   materialización point-in-time      │
│ pricing-engine    (Py + Rust)     simulación MC, modelos, calibración│
│ value-detector    (Rust)          devig, edge, z, filtros, tiers     │
│ staking-service   (Py, CVXPY)     Kelly bayesiano y conjunto         │
└──────────────────────────────────────┬───────────────────────────────┘
                                       │  signals.v1
┌──────────────── APLICACIÓN ──────────▼───────────────────────────────┐
│ api-gateway (GraphQL + REST, Node)   │ alert-service (Go)            │
│ bankroll-service (Py)                │ tracker-service (Py)          │
│ ws-hub (Go, WebSocket)               │ auth (Auth0/Keycloak, OIDC)   │
└──────────────────────────────────────┬───────────────────────────────┘
                                       ▼
        web (Next.js 15 / React 19 / TanStack Query)  ·  PWA  ·  Expo (Fase 2)
```

## 5.2 Elección de tecnologías y justificación

| Componente | Tecnología | Por qué |
|---|---|---|
| `odds-ingestor` | **Go** | Miles de conexiones concurrentes, GC predecible, latencia p99 estable. Python con asyncio no sostiene el fan-out de streams a 200 ms |
| `value-detector` | **Rust** | Evalúa ~15.000 combinaciones por tick. Debe ejecutarse en < 50 ms; es el camino crítico |
| `pricing-engine` | **Python + núcleo Rust (PyO3)** | El ecosistema científico es Python; la simulación Monte Carlo de 200.000 iteraciones se escribe en Rust y se expone como módulo |
| Inferencia bayesiana | **NumPyro** (JAX) sobre Stan | SVI para la actualización diaria (segundos), NUTS para el refit semanal; JAX aprovecha GPU si se necesita |
| Bus de eventos | **Redpanda** | API Kafka sin ZooKeeper/JVM; menor latencia de cola y mucho menor coste operativo a esta escala |
| Series temporales | **TimescaleDB** | Hipertablas, compresión nativa (~10×) y agregados continuos sobre SQL estándar; evita introducir una BD propietaria |
| OLAP | **ClickHouse** | Backtests que barren 10⁹ filas de cuotas históricas en segundos |
| Orquestación | **Dagster** | Activos versionados con verificación de frescura y linaje — encaja con la exigencia point-in-time mejor que Airflow |
| Frontend | **Next.js 15 + TanStack Query + Zustand** | SSR para SEO de páginas públicas, cliente reactivo para el board en vivo |
| Gráficos | **Visx / D3 + SVG** | Control total sobre las especificaciones de marcas del [doc 03](03-interfaz-y-experiencia-de-usuario.md#34-especificación-de-gráficos); las librerías de alto nivel imponen dobles ejes y paletas cicladas |
| Infra | **Kubernetes (EKS) + Terraform + ArgoCD** | GitOps, autoescalado por carga de jornada (sábado 16:00 ≠ martes 04:00) |

## 5.3 Presupuesto de latencia (camino crítico)

Objetivo extremo a extremo **p95 < 800 ms** desde que la casa cambia el precio
hasta que el usuario ve la alerta.

| Etapa | Presupuesto p95 | Notas |
|---|---|---|
| Proveedor → `odds-ingestor` | 250 ms | Fuera de nuestro control; se mide y se audita por proveedor |
| Normalización + resolución de entidad | 15 ms | Caché de mapeos en memoria; sin acceso a BD en el camino caliente |
| Publicación en Redpanda | 10 ms | `acks=1`, compresión LZ4 |
| `value-detector` (devig + edge + filtros) | 50 ms | Rust, sin asignaciones en el bucle caliente; probabilidades precalculadas |
| `staking-service` | 30 ms | Kelly individual en caliente; el conjunto se recalcula de forma asíncrona |
| `alert-service` → WebSocket | 40 ms | |
| Render en cliente | 100 ms | Actualización incremental, sin re-render global |
| **Margen de reserva** | **305 ms** | |

**Clave de diseño:** las probabilidades del modelo **no se calculan en el camino
caliente**. Se precalculan y cachean por evento, y solo se recalculan cuando
cambia una feature material (alineación, clima, lesión). Un tick de cuota
dispara únicamente aritmética de comparación, no una simulación Monte Carlo.

## 5.4 Estrategia de reprecio

| Disparador | Acción | Latencia |
|---|---|---|
| Alta de fixture (T−7 d) | Precio inicial completo | Batch nocturno |
| Actualización diaria de ratings | Reprecio de todos los fixtures abiertos | Batch, 04:00 UTC |
| **Alineación confirmada** | **Reprecio inmediato de prioridad máxima** | < 5 s |
| Noticia de lesión confirmada por 2 fuentes | Reprecio inmediato | < 10 s |
| Actualización meteorológica (T−6 h, T−2 h) | Reprecio de mercados sensibles | < 60 s |
| Designación arbitral | Reprecio de tarjetas y faltas | < 60 s |
| *Steam move* detectado | **No reprecia el modelo.** Recalcula la referencia de mercado y revalúa filtros | < 2 s |

La última fila es una decisión deliberada: un movimiento de mercado no es
información nueva sobre el partido, es información sobre el mercado. Dejar que
mueva el modelo lo convierte en un seguidor del mercado y destruye toda
capacidad de detectar valor.

## 5.5 Escalado y capacidad

| Magnitud | v1.0 |
|---|---|
| Eventos cubiertos por día | ~350 |
| Snapshots de cuota por día | ~25 M |
| Combinaciones evaluadas por día | ~15 M |
| Señales publicadas por día | 40–120 |
| Crecimiento del almacenamiento | ~8 GB/día sin comprimir; ~0,9 GB con compresión Timescale |
| Usuarios concurrentes objetivo | 5.000 (v1), 50.000 (v2) |

- Escalado horizontal de `odds-ingestor` particionando por casa.
- `value-detector` particionado por `event_id` (paralelismo natural, sin estado
  compartido).
- `ws-hub` escala por conexiones con Redis Pub/Sub como bus de fan-out.
- Autoescalado por calendario deportivo, no solo por CPU: la carga es predecible
  con días de antelación.

## 5.6 Fiabilidad y degradación elegante

| Fallo | Comportamiento |
|---|---|
| Caída de un proveedor de cuotas | Se continúa con los demás; las señales que dependían de él se retiran y se marca el descenso de cobertura en la UI |
| Caída de Pinnacle y Betfair a la vez | **Se suspende la publicación de señales.** Sin referencia sharp no hay medición fiable de valor. El sistema lo dice explícitamente |
| Caída del `pricing-engine` | Se sirven los últimos precios cacheados con marca de antigüedad; pasados 30 min se suspende la publicación |
| Datos de fixture incompletos | Ese fixture no se pone a precio (`BLOCKED_*`), el resto sigue |
| Deriva de calibración | *Kill switch* automático del modelo afectado; paso al modelo campeón anterior |

`RF-ARQ-01` — **Ninguna señal se publica con datos degradados sin marcarlo en la
propia señal.** Es preferible un panel vacío a un panel engañoso.

## 5.7 Observabilidad

- **Trazas:** OpenTelemetry extremo a extremo, con `trace_id` que sigue un tick
  de cuota desde el ingestor hasta la notificación push.
- **Métricas (Prometheus/Grafana):** latencia por etapa, frescura por proveedor,
  tasa de señales por tier, ECE móvil, CLV móvil, tasa de resolución de
  entidades, cobertura de fixtures.
- **Cuadro de mando de datos:** frescura, completitud y tasa de validaciones
  fallidas por fuente. Es el panel que mira el equipo de guardia.
- **Alertas operativas:** frescura > SLA, cobertura < 95 %, ECE > 0,05,
  CLV de 200 apuestas < 0, cola de resolución de entidades > 20.
- **Errores:** Sentry en cliente y servidor.
- **Auditoría:** cada señal publicada persiste su `pricing_run_id`, versión de
  modelo, `sim_seed`, vector de features y precios de entrada. **Una señal debe
  poder reconstruirse íntegramente seis meses después.**

## 5.8 Seguridad y privacidad

| Ámbito | Medida |
|---|---|
| Autenticación | OIDC (Auth0/Keycloak), MFA obligatorio para cuentas con bankroll > 10.000 € |
| Autorización | RBAC con ámbitos por organización (soporte a sindicatos) |
| Datos en reposo | Cifrado de disco; columnas de apuestas y bankroll con cifrado a nivel de aplicación (AES-GCM, claves en KMS) |
| Datos en tránsito | TLS 1.3 obligatorio; mTLS entre servicios internos |
| Credenciales de proveedores | AWS Secrets Manager con rotación; jamás en variables de entorno de imagen |
| **Credenciales de casas de apuestas** | **Nunca se solicitan ni se almacenan.** El producto no accede a las cuentas del usuario. La única excepción es la clave de API de exchange, opcional, cifrada por usuario y revocable |
| RGPD | Base legal documentada, exportación y supresión de datos en < 30 días, minimización, retención de 24 meses para datos de apuestas |
| Datos de terceros | Cumplimiento de los términos de licencia de cada proveedor; prohibida la redistribución de datos crudos de Opta/Sportradar |
| Rate limiting | Por usuario y por IP en la API pública; protección anti-scraping del propio producto |

## 5.9 Entornos y despliegue

| Entorno | Propósito |
|---|---|
| `dev` | Local con Docker Compose; datos sintéticos y *fixtures* grabados de proveedores |
| `staging` | Réplica de producción con feeds reales en modo sombra; **sin publicar señales a usuarios** |
| `prod` | Producción |
| `backtest` | Aislado, con acceso de solo lectura al lago de datos; sin conectividad a servicios de usuario |

- CI (GitHub Actions): lint, tipos (mypy estricto, `clippy`), tests unitarios,
  tests de contrato de proveedores, **tests de propiedad sobre el motor de
  probabilidades** (las probabilidades de un mercado suman 1; el devigging es
  monótono; Kelly nunca devuelve `f > 1`; la matriz de simulación es coherente
  entre mercados).
- CD: ArgoCD con despliegue canario y *rollback* automático por SLO.
- **Puerta de calidad de modelo en CI:** ningún modelo se promociona sin superar
  los umbrales de RPS, ECE y CLV en el fold de validación más reciente. La
  promoción de modelos es un *pull request* con su informe de métricas adjunto.
