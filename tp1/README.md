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
- **Tamaño:** 50.000 filas × 15 columnas (49.580 filas después de la limpieza).
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
| `lead_tramo` | Tramo de anticipación (0-15, 16-30, 31-60, … , +365 días). |

## Metodología

El desarrollo completo está en [`tp1.ipynb`](tp1.ipynb), hecho en Python con pandas, numpy, matplotlib, seaborn y scipy.

1. **Carga:** lectura del CSV en UTF-8.
2. **Limpieza:**
   - **Nulos:** no hay nulos explícitos. El valor `(not set)` en `booking_origin` (84 filas) es un nulo encubierto y se reemplazó por `NaN`.
   - **Duplicados:** se eliminaron 420 filas idénticas (0,84 %).
   - **Inconsistencias:** se validaron rangos y categorías (pasajeros > 0, horas de 0 a 23, binarias en 0/1, formato de ruta, etc.) y ninguna fila incumplió esas reglas. Se unificaron países duplicados o mal codificados: `Czechia`/`Czech Republic` y `R�union` → `Réunion`. En los viajes de solo ida la estadía se pasó a `NaN`, porque no tiene sentido. Se documentó una anomalía del dataset: `length_of_stay` no tiene valores entre 7 y 16 días.
   - **Tipos de datos:** las columnas de texto se convirtieron a `category` y `flight_day` a categoría ordenada.
3. **Análisis exploratorio:**
   - Tipos de variables y distribución de cada variable numérica.
   - Outliers con el criterio IQR (Tukey). Se conservaron porque son valores posibles, no errores. Los gráficos se recortan al percentil 99.
   - Histogramas con KDE de la duración del vuelo y del costo por pasajero, que comparan reservas completadas y no completadas.
   - Por hipótesis: histogramas con KDE, boxplots, scatterplots, gráficos de barras y de líneas, y matriz de correlación.
   - La conversión se compara siempre como **tasa** (%), porque la variable objetivo está desbalanceada. Para comparar entre rutas o países se exigió un mínimo de reservas (20, 30 o 100, según el gráfico), porque con pocas reservas el porcentaje es inestable.
   - H4 se analizó **ruta por ruta**: solo rutas vendidas por ambos canales, comparando la mediana del costo por pasajero.
4. **Pruebas estadísticas:**
   - **Chi-cuadrado** de independencia para H1, H2 y H3.
   - **Wilcoxon** pareado por ruta para H4.

## Conclusiones y hallazgos relevantes

| Hipótesis | Resultado | Evidencia principal |
|---|---|---|
| **H1** | ✅ Se cumple parcialmente (efecto chico) | Conversión de 17,5 % con 0-15 días de anticipación contra 13-14,5 % desde los 60 días, donde se estabiliza. Mediana de anticipación: 46 días en las completadas y 51 en las no completadas. Chi² significativo (p ≈ 5·10⁻¹⁴). |
| **H2** | ✅ Se cumple | La conversión sube con cada extra: 10,7 % → 15,0 % → 16,0 % → 18,5 % (0 a 3 extras). El equipaje extra es el adicional más asociado (16,7 % contra 11,5 %). Chi² significativo. |
| **H3** | ✅ Se cumple con claridad | La conversión va de 5,0 % (Australia) a 34,5 % (Malasia), y la diferencia se repite ruta por ruta. Chi² significativo (p ≈ 0). |
| **H4** | ❌ Se rechaza | En la misma ruta, Mobile e Internet cobran prácticamente lo mismo (diferencia mediana +1,3 %). Mobile es **más caro** en 88 de 144 rutas. Wilcoxon p = 0,99. |

### Hallazgos

1. **El país de origen es el factor que más diferencia la conversión.** Australia es el mercado más grande (36 % de las reservas) y uno de los que peor convierte (5 %), mientras que Malasia convierte casi 7 veces más. Mejorar la conversión en Australia tendría el mayor impacto sobre el total.
2. **Los adicionales están asociados a completar la reserva**, sobre todo el equipaje extra. Es una asociación, no necesariamente una causa: quien ya decidió viajar es quien más configura extras.
3. **La anticipación influye poco.** Las compras de último momento convierten algo más, pero desde los 60 días la conversión se mantiene estable.
4. **El canal no cambia el precio:** en la misma ruta, Mobile no es más barato que Internet.
5. **Los vuelos largos convierten menos.** Los vuelos de más de 8 horas convierten 10,7 %, contra 18,5 % de los más cortos. Esto ayuda a explicar H3: Australia reserva sobre todo vuelos de ~9 h (mediana 8,6 h) y Malasia de ~6,6 h. Las reservas completadas también son levemente más baratas (mediana del costo por pasajero 670 contra 701).
6. Ninguna variable numérica por sí sola tiene una correlación fuerte con completar la reserva (todas por debajo de 0,15 en valor absoluto). La conversión depende de una combinación de factores.

### Limitaciones

- Faltan fechas calendario y el motivo de la no conversión.
- `length_of_stay` tiene un hueco sin datos entre 7 y 16 días.
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
pip install pandas numpy matplotlib seaborn scipy jupyter
jupyter notebook tp1.ipynb
```
