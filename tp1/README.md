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

| Hipótesis | Resultado | Evidencia principal |
|---|---|---|
| **H1** | ✅ Se cumple parcialmente (efecto chico) | Conversión de 16,5 % con 0-29 días de anticipación contra 12,6-16,0 % desde los 60 días, sin tendencia clara. El 37 % de las completadas se compró con menos de 30 días, contra el 33 % de las no completadas (mediana 46 vs 51 días). |
| **H2** | ✅ Se cumple | La conversión sube con cada extra: 10,8 % → 15,0 % → 15,8 % → 18,7 % (0 a 3 extras). El efecto se repite dentro de las rutas: en 44 de 51 rutas la conversión con 3 extras supera a la de 0 extras. El equipaje extra es el adicional más asociado (16,6 % contra 11,6 %). |
| **H3** | ✅ Se cumple con claridad | La conversión va de 5,0 % (Australia) a 34,6 % (Malasia), y la diferencia se repite ruta por ruta. |
| **H4** | ❌ Se rechaza | En la misma ruta, Mobile e Internet cobran prácticamente lo mismo (diferencia mediana +0,7 %). Mobile es **más caro** en 80 de 145 rutas y en 8 de las 10 rutas principales. |

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