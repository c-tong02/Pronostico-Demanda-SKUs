# Pronóstico de Demanda a Nivel Tienda-Ítem

### Descripción del proyecto

El objetivo de este proyecto es predecir las ventas diarias de 50 productos en 10 tiendas durante los primeros 3 meses de 2018, generando un total de 45,000 predicciones individuales a nivel tienda-ítem. Se aborda un problema de forecasting supervisado sobre series de tiempo con 500 combinaciones paralelas y un horizonte de predicción de 91 días continuos. El propósito es habilitar la planificación de inventario y reposición de stock con anticipación suficiente para reducir quiebres de stock, sobre-inventario y costos asociados al capital de trabajo en operaciones de retail.

### Fuentes de dato

La base de datos utilizada para este proyecto es el dataset Store Item Demand Forecasting Challenge, disponible en Kaggle. Contiene 913,000 registros de ventas diarias entre 2013 y 2017, distribuidos en 10 tiendas y 50 ítems (500 series de tiempo paralelas). El conjunto de prueba contiene 45,000 registros correspondientes al período enero–marzo 2018, para los cuales se generan las predicciones.

- Dataset: <https://www.kaggle.com/competitions/demand-forecasting-kernels-only>

### Herramientas

- Python — EDA, ingeniería de variables y modelamiento
- LightGBM — gradient boosting para series de tiempo
- scikit-learn — métricas y preprocesamiento
- Pandas / NumPy — manipulación de datos y feature engineering
- Matplotlib / Seaborn — visualización
- Metodología CRISP-DM

### Limpieza / preparación de datos

En la fase inicial de preparación de datos, se hizo lo siguiente:

1. Carga e inspección de datos: 913,000 registros sin valores nulos ni inconsistencias. Dataset perfectamente balanceado (500 combinaciones × 365/366 días).
2. Análisis de tendencia y estacionalidad: crecimiento acumulado de ~20% entre 2013 y 2017, picos estacionales en julio–agosto y diciembre, y patrón semanal con fines de semana ~20% por encima del promedio.
3. Feature engineering sobre dataset combinado (train + test) para evitar data leakage — se generaron 39 features desde 3 columnas originales:
   - Lags de 91 a 365 días: el lag mínimo es siempre ≥ al horizonte de predicción (91 días) para garantizar zero data leakage en producción.
   - Rolling means y desviaciones estándar en 7 ventanas temporales (7, 14, 30, 60, 90, 180 y 365 días), shifteadas 91 días hacia atrás.
   - Expanding mean por combinación tienda-ítem para capturar el nivel base histórico global.
   - Variables de calendario: year, month, day, dayofweek, dayofyear, weekofyear, quarter, is_weekend, is_month_start, is_month_end.
   - Codificación cíclica (sin/cos) de mes y día de semana para respetar la circularidad del tiempo.
4. Split temporal estricto: 2013–2016 para entrenamiento (548,000 registros), 2017 para validación (182,500 registros). No se usa random split en series de tiempo.

### Modelamiento de datos

Se entrenó un modelo LightGBM global para las 500 series de tiempo simultáneamente, utilizando las 39 features de lag, rolling y calendario.

- Modelo: LightGBM Regressor (gradient boosting sobre árboles de decisión).
- Estrategia: modelo global único en lugar de 500 modelos individuales, aprovechando la regularización cruzada entre tiendas e ítems con patrones similares.
- Parámetros principales: learning_rate = 0.05, num_leaves = 64, feature_fraction = 0.8, bagging_fraction = 0.8, n_estimators = 720.
- Métricas de evaluación: RMSE, MAE y SMAPE (objetivo: SMAPE < 15%).

### Resultados

Los resultados del modelo en el conjunto de validación (año 2017) son:

1. **SMAPE = 12.02%** — cumple el criterio de éxito (objetivo < 15%). En promedio, el modelo se desvía un 12% del valor real, nivel aceptable para planificación de inventario.
2. **RMSE = 7.88 unidades** y **MAE = 6.08 unidades** — los residuos siguen una distribución aproximadamente normal centrada en 0, sin sesgo sistemático.
3. Las features más importantes del modelo fueron los lags de 364 y 365 días junto con el rolling mean de 365 días, confirmando que la demanda de un período está determinada principalmente por el mismo período del año anterior.
4. Las 45,000 predicciones generadas para enero–marzo 2018 mantienen los patrones estacionales históricos y son coherentes con el nivel base de cada tienda e ítem.

**Aplicación en operaciones:**

5. Las predicciones a nivel tienda-ítem-día permiten calcular órdenes de compra con 3 meses de anticipación por SKU y punto de venta, alimentar sistemas de procurement automatizado y dimensionar necesidades logísticas y de almacenamiento por tienda.
