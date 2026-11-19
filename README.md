# Análisis de taxis de Nueva York con PySpark
Proyecto de portfolio que simula un caso real de empresa: convertir los viajes de taxi de Nueva York en tablas listas para decidir **dónde y cuándo hay más demanda** y **qué zonas son más rentables**, comparando taxis amarillos y verdes.

> Estado: **Fases 1 a 4 completadas (exploración, Bronze, Silver y Gold)**. Siguiente: dashboard con mapa de zonas.

## Datos
- Fuente: [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) (datos públicos, formato Parquet).
- Tablas usadas: viajes de taxis amarillos, viajes de taxis verdes y el catálogo de zonas (`taxi_zone_lookup.csv`).
- Periodo analizado: `[enero y 2025]`, un mes de cada tipo de taxi. Más adelante se ampliará a un año.
- Los datos no se incluyen en el repositorio (carpeta `data/` ignorada). En los notebooks, la variable `BASE` apunta a la carpeta donde se guardan.

## Contenido del repositorio

| Fichero | Qué contiene |
|---|---|
| `N_yellow.ipynb` | Exploración (EDA) de los taxis amarillos con DuckDB |
| `N_green.ipynb` | Exploración (EDA) de los taxis verdes con DuckDB |
| `Capa bronze.ipynb` | Capa Bronze: carga de un mes de cada tipo de taxi con esquema unificado |
| `Capa_silver.ipynb` | Capa Silver: limpieza, banderas de calidad y unión con zonas |
| `Capa_gold.ipynb` | Capa Gold: tablas agregadas para las preguntas de negocio |

## Hallazgos principales del EDA

**Taxis amarillos**
- El tipo de pago 0 (Flex Fare) es un segmento aparte: distorsiona la distancia media (5,86 millas con él, unas 3,2 sin él).
- 144.118 viajes (4,1 %) tienen tarifa negativa: unos 62.700 con total también negativo (ajustes y devoluciones) y 81.402 con total positivo, casi todos del tipo 0.
- 63.037 viajes (1,8 %) tienen `total_amount` negativo.
- Los viajes con distancia 0 son, en su mayoría, fallos de medición.
- En el 32 % de los viajes, `total_amount` no coincide con la suma de sus componentes. Las diferencias son siempre importes exactos de tasas (2,50, 1,75, 0,75 y sus combinaciones), registradas de forma distinta según el proveedor. Decisión: usar `total_amount` para los ingresos y descartar el desglose de tasas.

**Taxis verdes**
- 1.836 viajes (3,8 %) llegan incompletos: `payment_type`, `passenger_count` y otras columnas nulas a la vez. Son viajes reales, más largos y caros, así que se conservan y se marcan.
- 153 viajes (0,3 %) con total negativo (anulaciones y ajustes).
- 43 viajes con fecha fuera del mes del fichero.
- Distancias absurdas: máximo de 84.731 millas; media 21,53 frente a mediana 1,74.
- No hay duplicados.

**Diferencia entre servicios**
- Los verdes recogen solo en zonas exteriores y norte de Manhattan; los amarillos mueven decenas de veces más viajes. Las comparaciones se hacen en proporciones y en las mismas zonas de recogida.
- `ehail_fee` está 100 % nula y se descarta.

**Reglas de calidad**

Principio: **marcar, no borrar**. Los datos crudos se conservan en Bronze, así que cada decisión es reversible. Los umbrales se definen como constantes en un único sitio.

| Regla | Columna | Criterio | Efecto |
|---|---|---|---|
| Ajuste / devolución | `is_adjustment` | `total_amount < 0` | Se excluye de demanda e ingresos |
| Tarifa negativa con total positivo | `fare_unreliable` | `fare_amount < 0` y `total_amount >= 0` | Usar `total_amount` en vez de la tarifa |
| Distancia no válida | `distance_valid` | `0 < trip_distance < 180` | Se excluye de análisis de distancia y velocidad |
| Pasajeros desconocidos | `passenger_unknown` | `passenger_count` nulo, 0 o ≥ 7 | Se conserva la fila; no se usa para análisis de pasajeros |
| Tipo de pago 0 (Flex Fare) | `payment_type` | `payment_type = 0` | Se muestra aparte; fuera del análisis de propinas |

### Preguntas de negocio (capa Gold)
- ¿Dónde y cuándo hay más demanda? (viajes por zona, hora y día de la semana)
- ¿Qué zonas dan más ingreso por minuto?
- ¿Dónde acaban los taxis y dónde se necesitan? (flujo neto: recogidas − destinos)
- ¿Qué zonas dan mejor propina? (solo pagos con tarjeta)
- ¿Cómo se paga en cada zona? (% tarjeta y efectivo)
- ¿Cuáles son las rutas origen-destino más importantes?
- ¿Cuánto pesan los aeropuertos y los viajes largos?
- ¿Dónde compiten verdes y amarillos? (% de verdes por zona) *(solo verdes)*
- ¿Se paran en la calle o se despachan? (`trip_type`) *(solo verdes)*

## Capa bronze
Los datos se guardan tal cual llegan, sin limpiar, pero con la misma estructura para los dos tipos de taxi:

- Nombres de columna en minúsculas y `tpep_`/`lpep_` unificados en `pickup_datetime` y `dropoff_datetime`.
- Un único esquema de 22 columnas de datos, con las propias de cada tipo de taxi rellenas con nulo en el otro.
- Columnas de trazabilidad: `taxi_type`, `source_file`, `ingested_at` y `year_month`.
- Parquet particionado por `taxi_type` y `year_month`.

## Capa Silver

Silver parte de Bronze y deja los datos listos para analizar. Principio: **marcar, no borrar**. Solo se eliminan las filas con fecha fuera del mes del fichero; el resto de problemas se marcan en columnas booleanas y cada análisis decide qué excluir. Los umbrales están definidos como constantes en un único sitio.

Resultado: **3.523.494 filas** (3.475.211 amarillos y 48.283 verdes).

| Regla | Columna | Criterio | Verdes | Amarillos |
|---|---|---|---|---|
| Fuera de mes (se elimina) | — | Recogida y llegada fuera de M−1→M, M→M o M→M+1 | 43 | 15 |
| Duplicados | - | Mismo viaje repetido | - | - |
| Ajuste / devolución | `is_adjustment` | `total_amount < 0` | 153 | 63.037 |
| Tarifa negativa con total positivo | `fare_unreliable` | `fare_amount < 0` y `total_amount >= 0` | 0 | 81.402 |
| Distancia no válida | `distance_valid` | `0 < trip_distance < 180` | - | - |
| Duración no válida | `duration_valid` | `0 < trip_minutes < [MAX_MINUTES]` | - | - |
| Pasajeros desconocidos | `passenger_unknown` | nulo, 0 o ≥ 7 | - | - |
| Tipo de pago Flex Fare | `is_flex_fare` | `payment_type = 0` | - | - |


## Capa Gold

Tablas agregadas, una por pregunta de negocio, con el tipo de taxi como columna para comparar amarillos y verdes. Todas excluyen los ajustes (`is_adjustment`) y aplican un mínimo de viajes donde hay cocientes o porcentajes.

| Tabla | Pregunta |
|---|---|
| `demanda_zona_hora_dia` | ¿Dónde y cuándo hay más demanda? |
| `ingreso_minuto_zona` | ¿Qué zonas dan más ingreso por minuto? |
| `flujo_neto_zona_hora` | ¿Dónde acaban los taxis y dónde se necesitan? |
| `propina_zona` | ¿Qué zonas dan mejor propina? (solo tarjeta) |
| `pago_zona` | ¿Cómo se paga en cada zona? |
| `rutas_origen_destino` | ¿Cuáles son las rutas más importantes? |
| `aeropuertos_viajes_largos` | ¿Cuánto pesan aeropuertos y viajes largos? |
| `cuota_verde_zona` | ¿Dónde compiten verdes y amarillos? |
| `tipo_viaje_verdes` | ¿Los verdes se paran en la calle o se despachan? |


# Hoja de ruta

(Se irá actualizando al lo largo del proyecto y se pueden incluir más fases)
- [x] Fase 1: exploración de datos con DuckDB
- [x] Fase 2: pipeline Bronze → Silver → Gold con PySpark (un mes de cada tipo de taxi)
  - [x] Bronze
  - [x] Silver
  - [x] Gold
- [ ] Dashboard con mapa de zonas
- [ ] Escalar de un mes a un año de datos y medir tiempos

