# Trabajo Práctico N° 1 — Análisis exploratorio de reservas aéreas

## Objetivo

Hacer un análisis exploratorio de datos (EDA) sobre un dataset de reservas de vuelos para responder cuatro hipótesis sobre qué factores se asocian a que una reserva **se complete o no** (conversión) y sobre diferencias de precio entre canales de venta. El trabajo incluye tratamiento y limpieza de los datos, visualizaciones y una conclusión con los hallazgos relevantes.

### Hipótesis

| | Hipótesis |
|---|---|
| **H1** | A mayor antelación de la compra, mayor probabilidad de que la reserva **no** se complete. |
| **H2** | A más adicionales solicitados (equipaje extra, asiento preferido, comidas), menor probabilidad de que la reserva no se complete. |
| **H3** | La conversión depende del país desde donde se reserva. |
| **H4** | Ante exactamente la misma ruta, es más barato reservar por **Mobile** que por **Internet** (web). |

## Contexto del dataset

- **Archivo:** `customer_booking.csv`, con reservas de vuelos hechas por clientes de una aerolínea. Para cada reserva trae el detalle del viaje, los adicionales solicitados, el costo y si se completó.
- **Fuente:** dataset público descargado de internet. <!-- TODO: completar con el link de origen y aclarar de dónde sale la columna booking_cost -->
- **Tamaño:** 50.000 filas × 15 columnas (48.792 filas después de la limpieza).
- **Variable objetivo:** `booking_complete`. Solo el **15 %** de las reservas se completa.
- **Limitaciones:**
  - No trae fechas calendario. La dimensión temporal se representa con el día de la semana, la hora del vuelo y los días de anticipación.
  - No informa el motivo por el que una reserva no se completa.

## Diccionario de datos

### Columnas originales

| Columna | Tipo | Descripción |
|---|---|---|
| `num_passengers` | Numérica discreta | Cantidad de pasajeros de la reserva (1 a 9). |
| `sales_channel` | Categórica | Canal de venta: `Internet` (web) o `Mobile`. |
| `trip_type` | Categórica | Tipo de viaje: `RoundTrip` (ida y vuelta), `OneWay` (solo ida), `CircleTrip` (circuito). |
| `purchase_lead` | Numérica discreta | Días entre la compra y la fecha del vuelo (anticipación). |
| `length_of_stay` | Numérica discreta | Días de estadía en el destino. No aplica a viajes de solo ida. |
| `flight_hour` | Numérica discreta | Hora de salida del vuelo (0 a 23). |
| `flight_day` | Categórica ordinal | Día de la semana del vuelo (`Mon` a `Sun`). |
| `route` | Categórica | Ruta: código IATA de origen + destino (por ejemplo, `AKLKUL` = Auckland → Kuala Lumpur). |
| `booking_origin` | Categórica | País desde donde se hizo la reserva. |
| `wants_extra_baggage` | Binaria | 1 si pidió equipaje extra. |
| `wants_preferred_seat` | Binaria | 1 si pidió asiento preferido. |
| `wants_in_flight_meals` | Binaria | 1 si pidió comidas a bordo. |
| `flight_duration` | Numérica continua | Duración del vuelo en horas. |
| `booking_complete` | Binaria | **Variable objetivo**: 1 si la reserva se completó. |
| `booking_cost` | Numérica continua | Costo total de la reserva (unidades monetarias del dataset). |

### Columnas derivadas

| Columna | Descripción |
|---|---|
| `total_extras` | Cantidad de adicionales pedidos (0 a 3). |
| `cost_per_passenger` | `booking_cost / num_passengers`. Sirve para comparar precios reduciendo el efecto del tamaño del grupo. |
| `lead_tramo` | Tramo de anticipación de 30 días (0-29, 30-59, … , 360-365). |

## Metodología

El desarrollo completo está en [`tp1.ipynb`](tp1.ipynb), hecho en Python con pandas, numpy, matplotlib y seaborn.

1. **Carga:** lectura del CSV con `encoding="ISO-8859-1"`.
2. **Limpieza:**
   - **Nulos:** no hay nulos explícitos. El valor `(not set)` en `booking_origin` (84 filas) es un nulo encubierto y se reemplazó por `NaN`; las filas se conservan.
   - **Duplicados:** hay 420 filas idénticas (0,84 %). Como el dataset no tiene un identificador de reserva, no se puede asegurar que sean errores (podrían ser ventas distintas), así que **se conservan**. Convierten de forma similar al resto (14 % vs 15 %), por lo que no cambian las conclusiones.
   - **Valores poco plausibles:** se eliminaron las compras con más de 365 días de anticipación (838) y las estadías de ida y vuelta de más de 180 días (370).
   - **Inconsistencias:** se validaron rangos y categorías (pasajeros > 0, horas de 0 a 23, binarias en 0/1, formato de ruta, etc.) y ninguna fila incumplió esas reglas. Se unificaron países duplicados o mal codificados: `Czechia`/`Czech Republic` y `Réunion` (que aparecía con un carácter dañado). En los viajes de solo ida la estadía se pasó a `NaN`, porque no tiene sentido. Se documentó una anomalía del dataset: `length_of_stay` no tiene valores entre 7 y 16 días.
   - **Tipos de datos:** las columnas de texto se convirtieron a `category` y `flight_day` a categoría ordenada.
3. **Análisis exploratorio:**
   - Clasificación de los tipos de variables.
   - Outliers con el criterio IQR (Tukey). Fuera de los casos poco plausibles eliminados en la limpieza, se conservaron porque son valores posibles, no errores. En el boxplot de anticipación el eje se recorta al percentil 99.
   - Histogramas con KDE de la anticipación (H1) y de los extras (H2), comparando reservas completadas y no completadas, y del precio relativo por canal (H4).
   - Por hipótesis: histogramas con KDE, boxplots, scatterplots, gráficos de barras y de líneas, y matriz de correlación.
   - La conversión se compara siempre como **tasa** (%), porque la variable objetivo está desbalanceada. Para comparar entre rutas o países se exigió un mínimo de reservas (10, 30 o 100, según el gráfico), porque con pocas reservas el porcentaje es inestable.
   - En el boxplot de H2 se calculó la conversión de cada ruta para 0, 1, 2 y 3 extras, usando solo las 51 rutas con 30+ reservas en los 4 grupos, para comparar siempre las mismas rutas.
   - H4 se analizó **ruta por ruta**: solo rutas vendidas por ambos canales, comparando la mediana del costo por pasajero.
   - Para variables con cola larga (como la anticipación) se usa la **mediana** y no la media: la media (≈80 días) queda arrastrada por unas pocas compras muy anticipadas, mientras que la mediana (50 días) representa mejor al cliente típico. Para la conversión se usa la **media**, porque en una variable 0/1 la media es la proporción de reservas completadas (distribución de Bernoulli).
4. **Inferencia estadística:** intervalos de confianza para la conversión y una prueba de hipótesis por cada hipótesis del trabajo, con las fórmulas de la prueba Z para proporciones y para la media y un nivel de significancia α = 0,05 (ver la sección siguiente).

## Inferencia estadística

Las reservas del dataset son una **muestra** de todas las reservas de la aerolínea (la **población**). La conversión que se
calcula con los datos es un **estadístico** (estimación puntual) del **parámetro** real, que no se conoce. La inferencia
permite decir con qué precisión se estima ese parámetro y si las diferencias observadas pueden deberse al azar.

### Intervalos de confianza para la conversión

Intervalo de confianza para una proporción, al 95 %:

> **p̂ ± 1,96 · √( p̂ · (1 − p̂) / n )**

Requisitos: muestra aleatoria y representativa, **np ≥ 5** y **n(1 − p) ≥ 5** (se cumplen en todos los grupos).

| Grupo | Reservas (n) | Conversión (p̂) | IC 95 % |
|---|---|---|---|
| Global | 48.792 | 15,0 % | [14,6 % ; 15,3 %] |
| Australia | 17.558 | 5,0 % | [4,7 % ; 5,3 %] |
| Malasia | 6.950 | 34,6 % | [33,5 % ; 35,7 %] |
| Australia — Internet | 16.060 | 5,3 % | [5,0 % ; 5,7 %] |
| Australia — Mobile | 1.498 | 1,4 % | [0,8 % ; 2,0 %] |

Ejemplo de cálculo con Australia: p̂ = 876 / 17.558 = 0,0499; error estándar = √(0,0499 × 0,9501 / 17.558) = 0,00164;
margen = 1,96 × 0,00164 = 0,0032 → **[4,7 % ; 5,3 %]**.

**Lectura:**
- *Con 95 % de confianza, la conversión real de Australia está entre 4,7 % y 5,3 %.* Un nivel de confianza del 95 %
  significa que, si se repitiera el muestreo muchas veces, el 95 % de los intervalos construidos contendría el valor real.
- Los intervalos de Australia y Malasia **no se superponen** (el máximo creíble de Australia, 5,3 %, está muy lejos del
  mínimo creíble de Malasia, 33,5 %): la diferencia entre mercados **no se explica por el azar del muestreo**.
- El intervalo de Australia Mobile es más ancho porque tiene muchas menos reservas: el tamaño de la muestra afecta el
  ancho del intervalo. Aun así, no se cruza con el de Australia por Internet: la diferencia entre canales también es real.

### Pruebas de hipótesis

Para cada hipótesis del trabajo se plantea una **hipótesis nula (H0)**, que representa "no hay efecto" y se asume
verdadera hasta que los datos muestren lo contrario, y una **hipótesis alternativa (Hₐ)**, que es lo que se quiere
demostrar. Se usa α = 0,05: si el **p-valor** (la probabilidad de observar un resultado igual o más extremo si H0 fuera
verdadera) es menor que 0,05, se rechaza H0.

> Para no confundir nombres: H1-H4 son las hipótesis del trabajo; H0 y Hₐ son las hipótesis estadísticas de cada prueba.

| Hipótesis del trabajo | H0 (no hay efecto) | Hₐ (lo que se quiere demostrar) | Prueba | Resultado | Decisión |
|---|---|---|---|---|---|
| **H1** — anticipación | La anticipación media de las completadas es igual o mayor que la de las no completadas | Las completadas se compran con **menos** anticipación (unilateral izquierda) | Z para la media (n > 30) | 75,3 vs 80,6 días; Z = −5,1; p ≈ 2·10⁻⁷ | Se rechaza H0 |
| **H2** — extras | La conversión con 3 extras es igual o menor que con 0 extras | Con 3 extras se convierte **más** (unilateral derecha) | Z para dos proporciones | 18,7 % vs 10,8 %; Z = 15,4; p ≈ 2·10⁻⁵³ | Se rechaza H0 |
| **H3** — país | La conversión de Australia es igual o mayor que la global (15 %) | Australia convierte **menos** (unilateral izquierda) | Z para una proporción | 5,0 % vs 15,0 %; Z = −37,0; p ≈ 0 | Se rechaza H0 |
| **H4** — canal | En la misma ruta, la diferencia media de precio Mobile − Internet es mayor o igual a 0 | Mobile es **más barato** (unilateral izquierda) | Z para la media de las diferencias por ruta (145 rutas) | Diferencia media +1,4 %; Z = +2,38; p = 0,99 | **No** se rechaza H0 |

**Lectura:**
- En H1, H2 y H3 las diferencias **no se explican por el azar**.
- En H4 no hay evidencia de que Mobile sea más barato. Una prueba **bilateral** (Hₐ: la diferencia es distinta de 0) da
  p = 0,017: hay diferencia, pero en sentido contrario, ya que Mobile es levemente **más caro** (IC 95 % de la diferencia:
  +0,2 % a +2,5 %).

**Advertencias:**
- **Significancia no es lo mismo que tamaño del efecto.** Con casi 49.000 reservas, hasta diferencias chicas resultan
  significativas: en H1 la diferencia es real, pero de solo 2 puntos (16,5 % vs 14,2 % de conversión con menos y con
  más de 30 días de anticipación). Por eso siempre se informa también el tamaño de la diferencia.
- **Nunca se prueba que H0 es verdadera:** en H4 no se concluye que los precios sean iguales, sino que no hay evidencia
  de que Mobile sea más barato.
- **No es un experimento (A/B test):** los clientes eligen su canal y sus extras, no se asignan al azar. Las pruebas
  muestran **asociación, no causalidad**.
- Los cálculos suponen que el dataset es representativo de todas las reservas de la aerolínea.
- Los errores posibles son el **tipo I** (rechazar H0 siendo verdadera; su probabilidad es α = 5 %) y el **tipo II**
  (no rechazar H0 siendo falsa).

## Conclusiones y hallazgos relevantes

### Insight de negocio

**Australia es el mercado más grande y el que peor convierte: es la mayor oportunidad de crecimiento.**

- Concentra el **36 %** de las reservas iniciadas, pero solo el **5,0 %** se completa, contra el **20,6 %** del resto
  de los países (4 veces menos).
- La caída es aún mayor en **Mobile**, donde Australia convierte apenas **1,4 %** (contra 5,3 % por Internet).

**Impacto:** si Australia convirtiera como el promedio del resto de los países, se completarían **≈2.700 reservas más**
con la misma demanda, **un 37 % más** que las ≈7.300 completadas hoy, sin invertir en atraer nuevos clientes.

Acción en Australia para optimizar la conversión. Analizar en qué paso del proceso se abandona
la reserva (precio final, medios de pago, experiencia en Mobile) y empezar por el canal Mobile, donde la brecha es mayor.

**Cómo validarlo (A/B test):** dividir al azar a los usuarios australianos de Mobile en dos grupos, A (proceso actual) y
B (proceso simplificado), y medir la tasa de conversión de cada uno (hoy 1,4 %). Con H0: "no hay diferencia" y
Hₐ: "B convierte más", una prueba de dos proporciones con α = 0,05 indicaría si la mejora es real. A diferencia del
análisis de este trabajo, la asignación aleatoria permitiría concluir causalidad.

| Hipótesis | Resultado | Evidencia principal |
|---|---|---|
| **H1** | ✅ Se cumple parcialmente (efecto chico) | Conversión de 16,5 % con 0-29 días de anticipación contra 12,6-16,0 % desde los 60 días, sin tendencia clara. El 37 % de las completadas se compró con menos de 30 días, contra el 33 % de las no completadas (mediana 46 vs 51 días). Prueba Z: p ≈ 2·10⁻⁷. |
| **H2** | ✅ Se cumple | La conversión sube con cada extra: 10,8 % → 15,0 % → 15,8 % → 18,7 % (0 a 3 extras). El efecto se repite dentro de las rutas: en 44 de 51 rutas la conversión con 3 extras supera a la de 0 extras. El equipaje extra es el adicional más asociado (16,6 % contra 11,6 %). Prueba Z: p ≈ 2·10⁻⁵³. |
| **H3** | ✅ Se cumple con claridad | La conversión va de 5,0 % (Australia, IC 95 % [4,7 ; 5,3]) a 34,6 % (Malasia, IC 95 % [33,5 ; 35,7]), y la diferencia se repite ruta por ruta. Prueba Z: p ≈ 0. |
| **H4** | ❌ No se cumple | En la misma ruta, Mobile e Internet cobran prácticamente lo mismo (diferencia mediana +0,7 %). Mobile es **más caro** en 80 de 145 rutas y en 8 de las 10 rutas principales. Prueba Z: p = 0,99 (no hay evidencia de que Mobile sea más barato). |

### Hallazgos

1. **El país de origen es el factor que más diferencia la conversión.** Australia es el mercado más grande (36 % de las reservas) y uno de los que peor convierte (5 %), mientras que Malasia convierte casi 7 veces más. Si Australia convirtiera como el promedio del resto de los países (20,6 %), se completarían ≈2.700 reservas más con la misma demanda, un 37 % más que las ≈7.300 completadas hoy. Es la mayor oportunidad del negocio.
2. **Los adicionales están asociados a completar la reserva**, sobre todo el equipaje extra. Es una asociación, no necesariamente una causa: quien ya decidió viajar es quien más configura extras.
3. **La anticipación influye poco.** Las compras de último momento convierten algo más, pero desde los 60 días la conversión oscila sin una tendencia clara.
4. **El canal no cambia el precio:** en la misma ruta, Mobile no es más barato que Internet.
5. Ninguna variable numérica por sí sola tiene una correlación fuerte con completar la reserva (todas por debajo de 0,11 en valor absoluto). La conversión depende de una combinación de factores.

### Limitaciones

- Faltan fechas calendario y el motivo de la no conversión.
- No hay un identificador de reserva, por lo que no se puede confirmar si las 420 filas duplicadas son errores o ventas distintas.
- `length_of_stay` tiene un hueco sin datos entre 7 y 16 días, y 84 reservas no tienen país de origen.
- Las relaciones encontradas son asociaciones: país, ruta y extras están mezclados entre sí y no se controlaron con un modelo multivariado.
- La inferencia supone que las reservas del dataset son representativas de todas las reservas de la aerolínea, y no se trata de un experimento con asignación aleatoria.

## Estructura del repositorio

```
├── README.md               ← este archivo
├── tp1.ipynb               ← notebook con el desarrollo completo (ejecutado)
├── customer_booking.csv    ← dataset
└── Trabajo Practico N1.pdf ← enunciado
```

## Cómo ejecutarlo

```bash
pip install pandas numpy matplotlib seaborn jupyter
jupyter notebook tp1.ipynb
```