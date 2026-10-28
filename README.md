# ​smart-inventory-replenishment-ml


                   ESTRUCTURA Y DICCIONARIO DE DATOS


1. TABLA: dim_products (Catálogo y Parámetros de Inventario)
Contiene los metadatos y parámetros financieros/logísticos calculados para cada 
familia de productos.

* product_id      (TEXT)   : Identificador único de la familia de productos.
* product_name    (TEXT)   : Nombre descriptivo de la familia de productos.
* category        (TEXT)   : Categoría general del producto (ej. 'Retail').
* unit_cost       (FLOAT)  : Costo unitario de compra estimado ($).
* lead_time_days  (INTEGER): Tiempo de entrega del proveedor en días (3, 5, 7 o 10 días).
* selling_price   (FLOAT)  : Precio de venta al público (calculado con un margen del +30%).


2. TABLA: dim_stores (Información de Tiendas)
Contiene los datos geográficos y la clasificación de los puntos de venta.

* store_nbr (INTEGER): Número identificador único de la tienda (1 a 54).
* city      (TEXT)   : Ciudad de Ecuador donde está ubicada la tienda.
* state     (TEXT)   : Provincia/Estado donde se ubica la tienda.
* type      (TEXT)   : Tipo/Tipo de formato de tienda (A, B, C, D, E).
* cluster   (INTEGER): Grupo o 'cluster' de tiendas con comportamiento de ventas similar.


3. TABLA: fact_oil_prices (Precios del Petróleo)
Registra la cotización del crudo WTI (West Texas Intermediate), indicador socioeconómico clave.

* date       (DATE) : Fecha del registro.
* dcoilwtico (FLOAT): Precio diario del barril de petróleo en USD (con imputación lineal para días faltantes).


4. TABLA: fact_daily_sales (Historial de Ventas Diarias)
--------------------------------------------------------------------------------
Tabla transaccional principal con los registros diarios por tienda y producto.

* id          (INTEGER): Identificador único del registro de venta.
* date        (DATE)   : Fecha de la transacción.
* store_nbr   (INTEGER): Clave foránea referenciando a 'dim_stores' (tienda).
* family      (TEXT)   : Clave foránea referenciando a 'dim_products' (familia de producto).
* sales       (FLOAT)  : Volumen total de ventas en unidades para esa combinación en el día.
* onpromotion (INTEGER): Número de artículos de esa familia que estaban en promoción ese día.


5. TABLA/CARACTERÍSTICAS: Time Series Features (Ingeniería de Variables)

Variables derivadas creadas dinámicamente para el entrenamiento del modelo de LightGBM.

* dayofweek        (INTEGER): Día de la semana (0 = Lunes, 6 = Domingo).
* month            (INTEGER): Mes del año (1 a 12).
* is_weekend       (INTEGER): Flag binario (1 si es fin de semana, 0 en otro caso).
* sales_lag_X      (FLOAT)  : Venta real histórica desfasada X días (lags: 7, 14, 21, 28, 30).
* rolling_mean_7   (FLOAT)  : Promedio móvil de ventas de los últimos 7 días.
* rolling_std_7    (FLOAT)  : Desviación estándar móvil de ventas de los últimos 7 días.
* rolling_mean_30  (FLOAT)  : Promedio móvil de ventas de los últimos 30 días.


6. ARCHIVO DE SALIDA: inventory_policy.csv (Políticas de Inventario)
Resultados generados tras la inferencia del modelo y aplicación de fórmulas de Supply Chain.

* family             (TEXT) : Familia de producto evaluada.
* daily_demand_pred  (FLOAT): Demanda promedio diaria predicha por el modelo (unidades/día).
* safety_stock       (FLOAT): Inventario de seguridad calculado para un nivel de servicio del 95% (unidades).
* reorder_point (ROP)(FLOAT): Punto de reorden que activa la orden de compra al proveedor (unidades).

                                  Diccionario de Variables

###. Variables de Ingeniería de Características (*Feature Engineering*)
Variables temporales y estadísticas calculadas dinámicamente para alimentar el modelo predictivo:

| Variable | Tipo de Dato | Descripción |
| :--- | :--- | :--- |
| `dayofweek` | Entero (0 - 6) | Día de la semana (0 = Lunes, 6 = Domingo). |
| `month` | Entero (1 - 12) | Mes del año. |
| `is_weekend` | Binario (0 / 1) | Indica si el día corresponde a fin de semana (1) o día laborable (0). |
| `sales_lag_X` | Flotante | Ventas históricas registradas hace `X` días (`lag_7`, `lag_14`, `lag_21`, `lag_28`, `lag_30`). |
| `rolling_mean_7` | Flotante | Promedio móvil de ventas de los últimos 7 días (excluyendo el día actual para evitar *Data Leakage*). |
| `rolling_std_7` | Flotante | Desviación estándar de las ventas de los últimos 7 días (mide la volatilidad a corto plazo). |
| `rolling_mean_30` | Flotante | Promedio móvil de ventas de los últimos 30 días (captura tendencias a mediano plazo). |

---

###  Resultados de Políticas de Inventario (`inventory_policy.csv`)
Métricas operativas calculadas a partir de las predicciones del modelo y fórmulas de Supply Chain:

| Variable | Tipo de Dato | Descripción |
| :--- | :--- | :--- |
| `family` | Texto | Categoría o familia de productos evaluada. |
| `daily_demand_pred` | Flotante | Demanda promedio diaria proyectada por el modelo de ML para los próximos días. |
| `safety_stock` | Flotante | **Stock de Seguridad:** Cantidad mínima de inventario de reserva necesaria para absorber variaciones inesperadas de la demanda o retrasos del proveedor (calculado para un Nivel de Servicio del 95%). |
| `reorder_point (ROP)`| Flotante | **Punto de Reorden:** Nivel crítico de inventario en almacén. Cuando el stock disponible cae a esta cifra, se debe emitir inmediatamente una nueva orden de compra al proveedor. |

---

## 🧮 Fórmulas Utilizadas

* **Stock de Seguridad:**
  $$\text{Safety Stock} = Z \times \sigma_{\text{error}} \times \sqrt{\text{Lead Time}}$$
  * *Donde $Z = 1.65$ (Nivel de servicio del 95%), $\sigma_{\text{error}}$ es la desviación estándar de los errores del pronóstico, y $\text{Lead Time} = 7 \text{ días}$.*

* **Punto de Reorden (ROP):**
  $$\text{ROP} = (\text{Demanda Promedio Diaria Predicha} \times \text{Lead Time}) + \text{Safety Stock}$$

                              References

_ https://pandas.pydata.org/docs/reference/api/pandas.Series.dt.dayofweek.html doumentation librery dayofweek
