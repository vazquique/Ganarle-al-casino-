# 03 — Interfaz y experiencia de usuario

## 3.1 Principios de diseño de la interfaz

El usuario de esta herramienta toma **decisiones financieras bajo presión de
tiempo** (una cuota con valor vive entre 30 segundos y 20 minutos). La interfaz
no es un panel de contemplación: es un instrumento de decisión.

1. **Una decisión por pantalla.** El panel principal responde a una sola
   pregunta: *¿en qué debo apostar ahora y cuánto?* Todo lo demás es secundario.
2. **La incertidumbre siempre visible.** Nunca se muestra "62 %" a secas. Se
   muestra "62 % (58–66)". Ocultar el intervalo produce exceso de confianza, que
   es el mecanismo por el que los usuarios se arruinan.
3. **Fricción proporcional al riesgo.** Filtrar es instantáneo; apostar por
   encima del stake recomendado exige confirmación explícita.
4. **Honestidad por defecto.** Si no hay señales, la pantalla lo dice con
   claridad y muestra cuándo se esperan las siguientes. Un panel que siempre
   tiene "oportunidades" es un panel que miente.
5. **Móvil primero.** El 70 % de las alertas se accionan desde el teléfono, de
   pie, en menos de un minuto.

---

## 3.2 Arquitectura de navegación

```
┌─ Value Board          (pantalla de aterrizaje: señales activas)
├─ Calendario           (todos los partidos, con o sin valor)
├─ Ficha de partido     (drill-down por evento)
├─ Mis apuestas         (tracker, CLV, bankroll)
├─ Rendimiento          (analítica personal: mapas de calor, segmentos)
├─ Alertas              (configuración de reglas y canales)
└─ Ajustes             (bankroll, límites, casas disponibles, responsabilidad)
```

---

## 3.3 Pantalla principal — *Value Board*

### 3.3.1 Maqueta

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  BANKROLL 4.812 €   ▲+2,1% 30d   │  CLV MEDIO +2,4%   │  ABIERTAS 6 · 310 €  │
│  ▁▂▃▄▄▅▆▇ (90d)                  │  ▁▃▂▄▅▄▆▅ (200 ap.)│  Exposición 6,4%     │
├──────────────────────────────────────────────────────────────────────────────┤
│ [Deporte ▾] [Liga ▾] [Mercado ▾] [Edge ≥ 2,5% ▾] [Casas: 6 ▾] [T− 24h ▾] [⭑] │   ← una sola fila de filtros
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ● A   Girona – Betis · LaLiga · hoy 21:00 (en 2 h 14 min)                    │
│        Over 2.5 goles                                                        │
│        ┌────────────┬──────────┬─────────┬────────┬───────────┬───────────┐  │
│        │ Cuota      │ Justa    │ Edge    │ Prob.  │ Stake     │ Casa      │  │
│        │ 2,05       │ 1,88     │ +8,9 %  │ 53,2 % │ 42 €      │ Bet365    │  │
│        │            │          │ z=2,71  │(50-56) │ (0,9% BR) │ lím. ~180€│  │
│        └────────────┴──────────┴─────────┴────────┴───────────┴───────────┘  │
│        ▸ Por qué: viento 4 km/h (↓), ambos equipos xG alto, árbitro permisivo │
│        ▸ Movimiento: 1,95 → 2,05 en 18 min · el mercado sharp va a 1,88       │
│        [ Ver partido ]  [ Registrar apuesta ]  [ Descartar ]  [ 🔔 Seguir ]   │
│                                                                              │
│  ● B   Brann – Bodø/Glimt · Eliteserien · mañana 18:00                        │
│        Hándicap asiático +0,25 Brann                                          │
│        1,96 │ 1,84 │ +6,5 % │ 51,0 % (46-56) │ 21 € │ Unibet                 │
│        ▸ Por qué: Bodø con 3 rotaciones confirmadas · 4 días menos de descanso│
│                                                                              │
│  ○ C   3 señales de confianza baja ocultas        [ Mostrar ]                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.2 Requisitos funcionales

| ID | Requisito |
|---|---|
| `RF-UI-01` | Ordenación por defecto: **EV total esperado** (`edge × stake recomendado`), no por edge bruto. Un 12 % de edge sobre un stake máximo de 5 € importa menos que un 3 % sobre 200 €. |
| `RF-UI-02` | Cada señal muestra **siempre** los seis campos núcleo: cuota disponible, cuota justa, edge, probabilidad con intervalo, stake sugerido y casa con límite estimado. |
| `RF-UI-03` | El *tier* de confianza se codifica con **color + icono + letra** (`● A`, `● B`, `○ C`), nunca solo con color. |
| `RF-UI-04` | Actualización en vivo por WebSocket. Si la cuota cae por debajo del umbral de valor, la fila se atenúa y muestra `Valor agotado — cuota ahora 1,92`; **no desaparece durante 60 s**, para que el usuario entienda qué pasó en lugar de ver filas evaporarse. |
| `RF-UI-05` | *Refetch* sin parpadeo: se mantiene el render anterior a opacidad reducida; nunca se muestra un esqueleto de carga sobre datos ya presentes. |
| `RF-UI-06` | Una única fila de filtros sobre todo el contenido; los filtros aplican a todas las vistas y persisten por usuario. Nunca filtros dentro de una tarjeta. |
| `RF-UI-07` | Estado vacío informativo: *"Sin señales que cumplan tus filtros. 34 partidos analizados en las últimas 6 h. Próxima ventana de valor prevista: alineaciones de LaLiga, 19:55."* |
| `RF-UI-08` | Virtualización de lista y presupuesto de render: ≤ 16 ms por actualización con 500 señales activas. |
| `RF-UI-09` | Acciones accesibles por teclado con atajos (`j/k` navegar, `Enter` abrir, `r` registrar, `x` descartar); objetivos táctiles ≥ 44 px en móvil. |

---

## 3.4 Especificación de gráficos

Reglas transversales aplicadas a **todos** los gráficos del producto:

- **Un solo eje Y por gráfico.** Prohibido el doble eje. Cuota y probabilidad
  son la misma magnitud en dos escalas: se representan **siempre en
  probabilidad implícita**, con el eje secundario de cuota solo como etiqueta de
  referencia del mismo eje.
- **El color sigue a la entidad, nunca a su posición.** Si el usuario filtra
  casas, las supervivientes conservan su tono.
- **Capa de interacción por defecto**: *crosshair* + tooltip en líneas, tooltip
  por marca en barras y celdas; el foco de teclado muestra lo mismo que el hover.
- **Gemelo tabular obligatorio.** Todo gráfico tiene un conmutador `Gráfico /
  Tabla`; ningún valor es accesible únicamente por color o por tooltip.
- **Marcas finas, rejilla en hairline sólido** (nunca discontinua), etiquetado
  directo selectivo (extremo, último punto, serie protagonista), nunca un número
  sobre cada punto.
- **Los colores de estado están reservados** a bueno/aviso/grave/crítico y no se
  reutilizan como color de serie.

### 3.4.1 Trayectoria de cuota (*line movement*) — gráfico insignia

**Trabajo del lector:** *¿Estoy por delante o por detrás del mercado, y hacia
dónde va el precio?*

- **Forma:** línea temporal, eje X desde apertura hasta saque inicial.
- **Trabajo del color:** **énfasis**, no categórico. La casa objetivo va en el
  tono de acento; el resto de casas en gris de de-énfasis. Es el caso de libro
  del anti-patrón "ocho tonos cuando la historia es una sola serie".
- **Series (máximo 4 con color propio):**
  1. Cuota de la **casa objetivo** (acento, etiquetada directamente en su
     extremo).
  2. **Línea justa del mercado sharp** (Pinnacle/Betfair devigged) — trazo de
     2 px en gris oscuro; es la referencia.
  3. **Probabilidad del modelo** con **banda de intervalo creíble al 90 %**
     (área translúcida del mismo tono, sin borde).
  4. Resto de casas: gris claro, sin leyenda individual, accesibles por tooltip.
- **Anotaciones sobre el eje temporal:** publicación de alineaciones, noticia de
  lesión, *steam move* detectado, y **el punto de entrada del usuario** si ya
  apostó (marcador con anillo de 2 px del color de superficie).
- **Prohibido:** doble eje cuota/probabilidad; rejilla discontinua; una serie por
  cada una de las 20 casas.

### 3.4.2 Mapa de calor de rentabilidad

**Trabajo del lector:** *¿Dónde gano y dónde pierdo dinero?*

- **Forma:** heatmap de rejilla, filas = liga, columnas = mercado (o franja
  temporal de apuesta).
- **Trabajo del color:** **divergente** — la magnitud representada (CLV medio o
  yield) tiene polaridad natural en torno a cero. Dos tonos opuestos
  (frío/cálido) con **punto medio gris neutro**. Nunca arcoíris, nunca un tono
  en el centro.
- **Celda:** muestra el valor; el tamaño de muestra `n` modula la **opacidad**
  (un segmento con n = 7 se ve casi transparente, señalando visualmente que no
  es interpretable). Las celdas con `n < 30` se rayan con textura y su tooltip
  advierte *"muestra insuficiente"*.
- **Máximo 7 clases de color.** Por encima, tabla.
- **Escala legendada** siempre presente, con los extremos etiquetados.

### 3.4.3 Curva de bankroll con banda de varianza

**Trabajo del lector:** *¿Esto es una mala racha normal o mi sistema está roto?*

Es el gráfico de mayor valor psicológico del producto.

- **Forma:** línea (bankroll real) sobre **banda de simulación Monte Carlo**
  (percentiles 5–95 de 10.000 trayectorias simuladas con el edge y la varianza
  estimados del usuario), más la mediana simulada como línea de referencia fina.
- **Trabajo del color:** un tono + gris. La banda es área translúcida neutra; la
  trayectoria real es el acento.
- **Lectura explícita en texto bajo el gráfico**, no dejada a la interpretación:
  > *"Tu bankroll está en el percentil 23 de las trayectorias simuladas. Es un
  > resultado normal: el 23 % de los recorridos con tu mismo edge van peor en
  > este punto. Tu CLV sigue en +2,4 %, lo que indica que la selección de
  > apuestas sigue siendo correcta."*
- Si la trayectoria sale de la banda **y** el CLV se ha vuelto negativo, el
  sistema muestra una alerta de estado `grave` con icono y texto: el problema no
  es la varianza, es el modelo o la ejecución.

### 3.4.4 Fila de KPIs (*stat tiles*)

Cuatro tarjetas, no un gráfico de barras agrupadas: **bankroll actual**, **CLV
medio**, **yield**, **exposición abierta**. Cada una con valor (cifra
proporcional, no tabular), delta respecto al periodo anterior y *sparkline*.
El bankroll actúa como **cifra protagonista** (≥ 48 px, misma tipografía sans
del resto, nunca serif ni display).

### 3.4.5 Medidor de exposición

Ratio contra un límite → **medidor** de pista única con el mismo tono, no un
gráfico de tarta de dos porciones. Umbrales de aviso a 70 % y 90 % del límite
configurado, con icono y etiqueta.

### 3.4.6 Diagrama de fiabilidad (calibración) — vista avanzada

Probabilidad predicha vs frecuencia observada, con diagonal de referencia en
hairline, barras de error de Wilson por bin y `n` en el tooltip. Es la prueba
que el usuario avanzado exigirá para confiar en el sistema; ocultarla sería un
error de producto.

### 3.4.7 Distribución de resultados simulados (ficha de partido)

Matriz de marcadores exactos como heatmap **secuencial** (un solo tono,
más-oscuro-es-más-probable), con las celdas del mercado seleccionado resaltadas
por anillo. Hace tangible de dónde sale la probabilidad.

---

## 3.5 Explicabilidad — *"¿por qué esta apuesta?"*

Sin esto, el usuario no ejecuta las señales incómodas (que suelen ser las
buenas) y sí ejecuta las que coinciden con su intuición (que suelen ser las
malas).

Cada señal incluye una explicación en dos niveles:

**Nivel 1 — una frase, generada por plantilla a partir de los 3 factores SHAP
dominantes:**
> *"Valor principalmente porque Bodø/Glimt confirma 3 rotaciones respecto al XI
> previsto y llega con 4 días menos de descanso; el mercado aún no lo ha
> incorporado por completo."*

**Nivel 2 — desglose expandible:**
- Gráfico de **barras horizontales divergentes** con la contribución SHAP de los
  8 factores principales, ordenadas por magnitud, centradas en cero (color
  divergente: empuja a favor / en contra).
- Comparativa `probabilidad de mercado → aportación del modelo → probabilidad
  final`, con el peso de mezcla `w_m` visible.
- Enlace a la ficha de partido con los datos crudos.

`RF-UI-10` — **Prohibidas las explicaciones generadas sin trazabilidad al
modelo.** Toda frase mostrada debe derivar de valores SHAP o de reglas
deterministas registradas, nunca de texto libre no anclado.

---

## 3.6 Sistema de alertas

| Canal | Uso | Latencia objetivo |
|---|---|---|
| WebSocket en app | Actualización del board | < 800 ms |
| Push (PWA / móvil) | Tier A y B según reglas del usuario | < 2 s |
| Telegram bot | Opcional, usuarios avanzados | < 2 s |
| Email | Solo resúmenes diarios | — |

**Reglas de alerta configurables** (editor de reglas, no un interruptor global):

```yaml
regla:
  nombre: "Valor alto en LaLiga pre-alineaciones"
  condiciones:
    deporte: futbol
    competiciones: [ESP_1, ESP_2]
    mercados: [1X2, OU_GOLES, AH]
    edge_min: 3.0
    tier_min: B
    casas: [bet365, williamhill, betfair]
    ventana: "T-6h .. T-30min"
    stake_minimo_ejecutable: 20
  canal: [push, telegram]
  limites:
    max_alertas_hora: 6
    silencio: "00:00-08:00 Europe/Madrid"
    cooldown_por_evento_min: 15
```

`RF-UI-11` — **Control de fatiga obligatorio.** Un máximo configurable de
alertas por hora, con `cooldown` por evento y ventana de silencio. Un sistema
que dispara 80 alertas al día entrena al usuario a ignorarlas y, peor, a apostar
compulsivamente. El valor por defecto es conservador: 6 alertas/hora, solo
tier A y B.

`RF-UI-12` — Toda alerta caduca visualmente: incluye la cuota en el momento del
disparo y, al abrirla, la cuota actual y si el valor persiste.

---

## 3.7 Ficha de partido (*match center*)

Todo lo que el modelo usó, expuesto para auditoría del usuario:

- Cabecera: equipos, competición, hora, estadio, **estado del dato** (alineación
  confirmada / probable, clima pronosticado, árbitro designado).
- Panel de modelo: probabilidades de todos los mercados con intervalo, matriz de
  marcadores, `λ` y `μ` estimados.
- Panel de forma: EWMA de npxG a favor y en contra, ajustada por rival
  (dos series categóricas, leyenda + etiquetado directo).
- Panel de contexto: alineaciones con impacto por ausencia, descanso, viaje,
  clima horario, perfil del árbitro.
- Panel de mercado: comparador de las mejores cuotas por mercado, margen de cada
  casa y trayectoria de cuota (§3.4.1).

---

## 3.8 Tracker de apuestas

| ID | Requisito |
|---|---|
| `RF-UI-13` | Registro en un clic desde la señal (precarga evento, mercado, cuota, stake sugerido), editable. |
| `RF-UI-14` | Registro manual y **importación CSV** con mapeo de columnas guardable. |
| `RF-UI-15` | **CLV calculado automáticamente** al cierre de cada evento, y liquidación automática del resultado. |
| `RF-UI-16` | Segmentación del rendimiento por liga, mercado, tier, casa, rango de cuota y hora del día, con `n` y **intervalo de confianza bootstrap siempre visibles**. |
| `RF-UI-17` | **Modo *paper trading*** con bankroll virtual, activado por defecto para cuentas nuevas durante 30 días o 100 apuestas. |
| `RF-UI-18` | Advertencia automática cuando el usuario segmenta hasta muestras no interpretables: *"12 apuestas no permiten concluir nada sobre este segmento."* |

---

## 3.9 Accesibilidad y rendimiento

- WCAG 2.2 AA: contraste ≥ 4,5:1 en texto, foco visible, navegación completa por
  teclado, `prefers-reduced-motion` respetado (sin animaciones de parpadeo en
  cambios de cuota; se usa un cambio de fondo suave y el valor con flecha).
- **Modo oscuro seleccionado, no invertido**: pasos de rampa propios validados
  contra la superficie oscura.
- Ningún estado se codifica solo por color: siempre color + icono + texto.
- Presupuesto de rendimiento: LCP < 1,8 s en 4G, INP < 200 ms, bundle inicial
  < 250 kB comprimido.
- Internacionalización: es-ES y en-GB en v1; formato de cuota configurable
  (decimal / fraccional / americana) con conversión en cliente.
