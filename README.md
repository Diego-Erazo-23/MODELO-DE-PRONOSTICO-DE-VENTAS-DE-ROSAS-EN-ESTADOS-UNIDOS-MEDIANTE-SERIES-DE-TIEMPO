# Modelo de Pronóstico de Ventas de Rosas en EE.UU. mediante Series de Tiempo

**Maestría en Inteligencia de Negocios y Ciencia de Datos — UDLA**  
**Autor:** Diego Mauricio Erazo Herrera  
**Año:** 2026

---

## Tabla de Contenidos

1. [Descripción del Proyecto](#descripción-del-proyecto)
2. [Estructura del Repositorio](#estructura-del-repositorio)
3. [Sobre los Datos](#sobre-los-datos)
4. [Descripción del Notebook](#descripción-del-notebook)
5. [Cómo Replicarlo](#cómo-replicarlo)
6. [Dependencias](#dependencias)
7. [Resultados Obtenidos](#resultados-obtenidos)
8. [Referencias](#referencias)

---

## Descripción del Proyecto

Este proyecto desarrolla un modelo analítico para el **pronóstico de la demanda de rosas ecuatorianas en el mercado de EE.UU**, utilizando la metodología de series de tiempo **SARIMAX**.

El caso de negocio corresponde a una comercializadora de flores ubicada en Miami como centro logístico. El problema central es la **alta estacionalidad de la demanda**, con picos pronunciados en San Valentín (semanas 4–6) y Día de la Madre (semanas 17–19), y caídas de las ventas en verano. Dado que las rosas son un producto perecedero, subestimar la demanda genera quiebres de stock en temporada pico, y sobreestimarla genera inventario dañado y costos de destrucción.

El proyecto compara tres modelos:

| Modelo | Tipo | MAPE | Descripción |
|--------|------|------|-------------|
| **M1** | SARIMA(3,1,3)(1,0,1)₅₂ | 23.72% | Benchmark sin variables exógenas |
| **M2** | SARIMAX(3,1,3)(1,0,1)₅₂ | 24.81% | SARIMAX con variable ordinal de festividades (replica Falatouri et al., 2022) |
| **M3** | SARIMAX(1,1,3)(1,0,1)₅₂ | **16.97%** | **Modelo propuesto**: 5 dummies individuales + costo logístico |

El modelo M3 logra una reducción de **6.75 puntos porcentuales** respecto al benchmark (28.5% de mejora relativa), cumpliendo el objetivo establecido de MAPE < 25%.

---

## Estructura del Repositorio

```
sarimax-rosas-eeuu/
│
├── README.md                          ← Este archivo
├── Modelo_SARIMAX_DEFINITIVO.ipynb    ← Notebook principal con todo el análisis
│
├── data/
│   └── README_datos.md                ← Explicación sobre la confidencialidad de los datos
│
└── outputs/
    ├── figura1_estacionalidad.png
    ├── figura2_serie_anual.png
    ├── figura3_costo_logistico.png
    ├── figura4_correlacion.png
    ├── figura5_acf_pacf.png
    ├── figura6_pronostico_vs_real.png
    ├── figura7_comparacion_mape.png
    ├── figura8_distribucion_errores.png
    └── figura9_coeficientes_m3.png
```

---

## Sobre los Datos

> ⚠️ **Los datos originales no están disponibles en este repositorio por razones de confidencialidad empresarial.**

### ¿Por qué no se publican los datos?

Las bases de datos utilizadas en este proyecto provienen de los **registros operativos internos de Naranjo Farms LLC**, una empresa privada ecuatoriana dedicada a la exportación y comercialización de rosas cortadas en el mercado de EE.UU.

La base de datos principal contiene **143.682 transacciones individuales** con información detallada a nivel de factura, incluyendo: clientes, volúmenes por variedad, precios de venta, rutas logísticas y costos de flete. La base exógena contiene información semanal de **costos logísticos internos** (flete aéreo Quito–Miami y handling).

Publicar estos datos implicaría:

- **Exposición de información comercial sensible**: volúmenes de venta por cliente y variedad que constituyen ventaja competitiva.
- **Revelación de estructura de costos logísticos**: las tarifas de flete negociadas con cargueras son información estratégica no pública.
- **Posible identificación de clientes**: los registros de facturación contienen datos que podrían permitir la identificación de la cartera de clientes de la empresa.
- **Compromiso contractual de confidencialidad**: los datos fueron cedidos exclusivamente para uso académico bajo acuerdo de no divulgación.

### ¿Qué necesito para replicar el análisis?

Para replicar el notebook con datos propios, necesitarás construir **dos archivos CSV** con la siguiente estructura:

#### Archivo 1: `historico_ventas_2021-2026.csv`

Debe contener al menos las siguientes columnas (los nombres deben coincidir exactamente):

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `Truck Date` | fecha (DD/MM/YYYY) | Fecha de la transacción |
| `Category` | texto | Categoría del producto. El modelo filtra: `Y` (rosas estándar), `AB` (spray roses), `AE` (rosas tinturadas) |
| `Total Units` | numérico entero | Cantidad de tallos vendidos en esa línea de factura |
| `Total Price` | numérico decimal | Valor total de la línea de factura en USD |
| `Invoice Number` | texto/numérico | Número de factura (para conteo de transacciones) |

#### Archivo 2: `Variables_exogenas.csv`

Debe contener exactamente 7 columnas en este orden:

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `AÑO` | entero | Año ISO (ej: 2021) |
| `SEMANA` | entero | Semana ISO del año (1–52) |
| `COSTO_FLETE_KG` | decimal | Costo del flete aéreo en USD/kg |
| `COSTO_HANDLING_KG` | decimal | Costo de handling en USD/kg |
| `COSTO_TOTAL_KG` | decimal | Suma de flete + handling (USD/kg) |
| `IMPUESTOS` | decimal | Tasa arancelaria vigente (no se usa en el modelo, pero debe estar presente en el CSV) |
| `TEMPORADA` | texto | Etiqueta de la semana: `VALENTIN`, `MADRES`, o `NORMAL` |

> **Nota sobre `TEMPORADA`:** Las semanas festivas están definidas según el análisis exploratorio: semanas 4–6 = `VALENTIN`, semanas 17–19 = `MADRES`, el resto = `NORMAL`. Esta codificación puede ajustarse según el calendario de tu mercado objetivo.

---

## Descripción del Notebook

El notebook `Modelo_SARIMAX_DEFINITIVO.ipynb` está organizado en **23 secciones numeradas**, más las celdas de figuras. A continuación se describe qué hace cada bloque:

### Bloque 0 — Librerías
Importa todas las dependencias necesarias: `numpy`, `pandas`, `matplotlib`, `seaborn`, `statsmodels` (SARIMAX, ADF, ACF/PACF, Ljung-Box) y `sklearn` (RMSE).

### Bloque 1 — Configuración global de matplotlib
Instala la fuente Arial en el entorno de Google Colab y configura el tamaño global de 12pt para todos los gráficos. Si se ejecuta localmente, usa DejaVu Sans como alternativa.

### Bloque 2 — Carga del archivo principal
Lee el CSV de ventas históricas y convierte la columna `Truck Date` a formato `datetime`. Imprime el número de registros y el rango de fechas.

### Bloque 3 — Filtro por categorías y construcción de la serie semanal
Filtra las categorías `Y`, `AB` y `AE` (rosas). Calcula el año y semana ISO de cada transacción. Agrega los tallos vendidos por semana ISO, obteniendo la serie temporal de **269 observaciones semanales** que alimentará los modelos.

### Bloque 4 — Incorporación de variables exógenas
Une el dataset de variables exógenas con la serie semanal usando la clave compuesta `AÑO + SEMANA ISO`. Crea la variable ordinal `FES` (0=NORMAL, 1=MADRES, 2=VALENTIN) para el modelo M2.

### Bloque 5 — Índice de estacionalidad
Calcula el índice de demanda relativa para cada semana del año (valor real / promedio global). Imprime los índices de las semanas festivas clave: sem. 5 = 3.935x, sem. 6 = 3.264x, sem. 18 = 2.717x.

### Figuras 1–4 — Análisis exploratorio visual

- **Figura 1 (Bloque entre 5 y 8b):** Matriz de correlación entre tallos semanales, costo logístico y temporada. Muestra r=0.63 entre tallos y festividad, r=0.38 entre tallos y costo logístico.
- **Figura 2:** Índice de estacionalidad semanal con barras coloreadas por festividad.
- **Figura 3:** Serie temporal anual 2021–2026, una línea por año, mostrando la repetibilidad del patrón estacional.
- **Figura 4:** Evolución del costo logístico total por kg con marcas de semanas festivas y la media histórica (2.397 USD/kg).

### Bloque 8b — Identificación y descripción formal de variables
Documenta todas las variables del modelo (tipo, descripción, índices históricos). Equivalente a la Tabla 5 del documento escrito.

### Bloque 9 — Preparación final para el modelo
Construye el índice de fechas en formato `DatetimeIndex` con frecuencia semanal (`W-MON`) necesario para que `statsmodels` procese correctamente la serie temporal.

### Bloque 10 — Prueba de estacionariedad ADF
Aplica el test de Dickey-Fuller Aumentado. Resultado: serie original p=0.098 (no estacionaria) → primera diferencia p≈0.000 (estacionaria). Esto fija **d=1** para todos los modelos.

### Figura 5 — ACF y PACF (Bloque después de 10)
Panel 2×2 con ACF y PACF de la serie original y su primera diferencia. El pico de ACF en rezago 52 (+0.536) confirma s=52. El PACF en rezago 52 (+0.155 > banda ±0.120) fija P=1 y Q=1.

### Bloque 11 — Justificación empírica de D=0
Compara AIC con D=0 (4112.8) vs D=1 (9835.8). La diferencia de +5723 puntos y la pérdida de 52 observaciones adicionales justifican D=0.

### Bloques intermedios — Auto-ARIMA y construcción de variables dummy
Ejecuta Auto-ARIMA para seleccionar los órdenes p y q óptimos por modelo. Construye las 5 variables dummy individuales (V_PRE, V_PEAK, M_PRE, M_PEAK, M_POST) y la variable LOG_c (costo logístico centrado en su media). Define los conjuntos de entrenamiento (217 semanas) y prueba (52 semanas).

### Bloque de diagnóstico de LOG_c
Analiza la colinealidad de la variable de costo logístico: correlación total r=+0.38 vs correlación en semanas normales r=+0.11. Compara MAPE con y sin LOG_c para justificar su retención.

### Bloque 18 — Modelo M1 (SARIMA benchmark)
Ajusta SARIMA(3,1,3)(1,0,1)₅₂ sin variables exógenas. Genera pronósticos para las 52 semanas de prueba. Calcula MAPE, RMSE, MAE y verifica ruido blanco con Ljung-Box.

### Bloque 19 — Modelo M2 (SARIMAX con FES ordinal)
Ajusta SARIMAX(3,1,3)(1,0,1)₅₂ con la variable ordinal de festividades, replicando la metodología de Falatouri et al. (2022). Resultado: MAPE=24.81%, **peor que M1**, confirmando que codificar igual semanas con índices muy distintos introduce sesgo sistemático.

### Bloque 20 — Modelo M3 (modelo propuesto)
Ajusta SARIMAX(1,1,3)(1,0,1)₅₂ con las 5 dummies individuales y LOG_c. Imprime todos los coeficientes estimados con sus p-valores y niveles de significancia. **MAPE=16.97%**, reducción de 28.5% respecto al benchmark.

### Bloque 21 — Tabla comparativa y MAPE segmentado
Resume los tres modelos en una tabla. Calcula el MAPE por segmento: semanas normales (13.95%), Día de la Madre (52.06%, dominado por la semana 19), San Valentín (27.15%, dominado por el pico histórico de Feb 2026). Cobertura APE<25%: 78.4% de las semanas.

### Bloque 22 — Horizonte de pronóstico de 12 semanas
Evalúa el MAPE a diferentes horizontes de pronóstico (1, 4, 8, 12 semanas). Verifica que el error se mantiene aceptable en las 12 semanas necesarias para el ciclo logístico completo Quito–Miami.

### Figuras 6–9 — Visualizaciones de resultados

- **Figura 6:** Pronóstico vs. ventas reales para las 52 semanas de prueba, con las tres curvas de modelo y marcas del shock arancelario 2025 y San Valentín 2026.
- **Figura 7:** Gráfico de barras comparando el MAPE de los tres modelos contra los umbrales objetivo (25%), bueno (20%) y excelente (10%).
- **Figura 8:** Histogramas del error porcentual absoluto para M1 y M3, mostrando cómo M3 concentra más semanas en rangos bajos.
- **Figura 9:** Coeficientes estimados de M3 con codificación por significancia estadística (verde/naranja/gris).

### Bloque 23 — Resumen ejecutivo final
Imprime un resumen completo con métricas, validación estadística, mejora vs benchmark, nota sobre LOG_c y tabla de verificación semanal con errores individuales.

---

## Cómo Replicarlo

### Opción A: Google Colab (recomendado)

1. Sube el notebook `Modelo_SARIMAX_DEFINITIVO.ipynb` a Google Colab.
2. Sube ambos archivos CSV a Google Drive en la siguiente ruta:
   ```
   Mi unidad/CLASES MAESTRIA/Proyecto MBD/
   ```
   Con los nombres exactos:
   - `historico_ventas_2021-2026.csv`
   - `Variables_exogenas.csv`
3. Ejecuta el notebook de arriba a abajo con `Runtime > Run all`.
4. En el **Bloque 1**, si el entorno no tiene Arial, se usará DejaVu Sans automáticamente.
5. Los archivos PNG de las figuras se guardarán en el directorio de trabajo de Colab.

> **Nota:** El Bloque 1 instala la fuente Arial con `apt-get`. Esto puede tomar 30–60 segundos la primera vez.

### Opción B: Jupyter local (Anaconda / venv)

1. Instala las dependencias (ver sección siguiente).
2. Clona o descarga este repositorio.
3. Coloca ambos archivos CSV en la misma carpeta que el notebook.
4. Modifica las rutas de carga en los **Bloques 2 y 4**:
   ```python
   # Bloque 2 — reemplaza esto:
   file_path = "/content/drive/My Drive/CLASES MAESTRIA/Proyecto MBD/historico_ventas_2021-2026.csv"
   # por esto:
   file_path = "historico_ventas_2021-2026.csv"

   # Bloque 4 — reemplaza esto:
   exog_path = "/content/drive/My Drive/CLASES MAESTRIA/Proyecto MBD/Variables_exogenas.csv"
   # por esto:
   exog_path = "Variables_exogenas.csv"
   ```
5. **Comenta o elimina** las líneas de montaje de Google Drive en el Bloque 0:
   ```python
   # from google.colab import drive        ← comentar esta línea
   # drive.mount('/content/drive')         ← comentar esta línea
   ```
6. **Comenta o elimina** el bloque de instalación de fuentes en el Bloque 1 (el bloque `subprocess.run`). La fuente Arial no estará disponible localmente a menos que la tengas instalada; el código la reemplaza automáticamente por DejaVu Sans.
7. Ejecuta el notebook celda por celda con `Shift+Enter` o completo con `Kernel > Restart & Run All`.

### Tiempos de ejecución esperados

| Bloque | Tiempo estimado |
|--------|-----------------|
| Carga y preprocesamiento (bloques 0–9) | < 1 minuto |
| Prueba ADF + ACF/PACF (bloques 10–11) | < 30 segundos |
| Auto-ARIMA (bloque de selección de parámetros) | 2–5 minutos |
| Ajuste M1 (bloque 18) | 30–60 segundos |
| Ajuste M2 (bloque 19) | 30–60 segundos |
| Ajuste M3 (bloque 20) | 30–60 segundos |
| Figuras y resumen final | < 1 minuto |
| **Total** | **~8–12 minutos** |

---

## Dependencias

```txt
numpy>=1.23
pandas>=1.5
matplotlib>=3.6
seaborn>=0.12
statsmodels>=0.13
scikit-learn>=1.1
```

### Instalación

**Con pip:**
```bash
pip install numpy pandas matplotlib seaborn statsmodels scikit-learn
```

**Con conda:**
```bash
conda install numpy pandas matplotlib seaborn statsmodels scikit-learn
```

**Versión de Python:** 3.8 o superior.

---

## Resultados Obtenidos

### Métricas del modelo propuesto (M3)

| Métrica | Valor |
|---------|-------|
| MAPE | **16.97%** |
| RMSE | 44.301 tallos/semana |
| MAE | 21.556 tallos/semana |
| AIC | 3.988 |
| Semanas con error < 25% | **78.4%** |

### Coeficientes del modelo M3 — SARIMAX(1,1,3)(1,0,1)₅₂

| Variable | Coeficiente estimado | p-valor | Significancia |
|----------|---------------------|---------|---------------|
| V_PEAK (sem. 5–6) | +255.000 tallos | < 0.001 | *** |
| M_PEAK (sem. 18) | +151.000 tallos | < 0.001 | *** |
| M_PRE (sem. 17) | +125.000 tallos | < 0.001 | *** |
| M_POST (sem. 19) | +70.000 tallos | < 0.05 | * |
| V_PRE (sem. 4) | ~0 | n.s. | — |
| LOG_c | +35.165 tallos/USD·kg⁻¹ | 0.115 | n.s.* |

> \* LOG_c no alcanza significancia estadística convencional (p=0.115), pero su retención mejora el MAPE en 1.75 puntos porcentuales fuera de muestra (Hyndman & Athanasopoulos, 2021).

### Validación estadística

- **Ljung-Box (lag 10):** p = 0.609 → residuos son ruido blanco ✔
- **Ljung-Box (lag 20):** p = 0.367 → residuos son ruido blanco ✔

---

## Referencias

- Altamirano, H. R., Ortega, M., & Páez, V. (2025). Exportaciones de flores ecuatorianas y su estacionalidad. *Ciencias Administrativas*, (25), 153. https://doi.org/10.24215/23143738E153
- Andrade, C. F. O., Herrera, P. A. L., & Bonilla, D. A. G. (2025). Google trends as an early signal in international flower trade. *Ornamental Horticulture*, 31, e312959. https://doi.org/10.1590/2447-536X.V31.E312959
- Eiglsperger, J., et al. (2024). Forecasting seasonally fluctuating sales of perishable products in the horticultural industry. *Expert Systems with Applications*, 249, 123438. https://doi.org/10.1016/j.eswa.2024.123438
- Falatouri, T., Darbanian, F., Brandtner, P., & Udokwu, C. (2022). Predictive Analytics for Demand Forecasting – A Comparison of SARIMA and LSTM in Retail SCM. *Procedia Computer Science*, 200, 993–1003. https://doi.org/10.1016/j.procs.2022.01.298
- Giri, A., & Giri, V. R. (2024). Forecasting cauliflower prices in Nepal. *Cogent Food & Agriculture*, 10(1). https://doi.org/10.1080/23311932.2024.2340155
- Guaita-Pradas, I., Rodríguez-Mañay, L. O., & Marques-Perez, I. (2023). Competitiveness of Ecuador's Flower Industry. *Sustainability*, 15(7), 5821. https://doi.org/10.3390/su15075821
- Herrera-Granda, I. D., et al. (2020). A Forecasting Model to Predict the Demand of Roses in an Ecuadorian Small Business. *Lecture Notes in Computer Science*, 12566, 245–258.
- Hyndman, R., & Athanasopoulos, G. (2021). *Forecasting: Principles and Practice* (3rd ed.). OTexts. https://otexts.com/fpp3/
- Petropoulos, F., et al. (2022). Forecasting: theory and practice. *International Journal of Forecasting*, 705–871. https://doi.org/10.1016/j.ijforecast.2021.11.001
- Zevallos-Aquije, A., et al. (2025). Improving Accuracy in Agricultural Forecasts. *Research on World Agricultural Economy*, 6(3), 273–290. https://doi.org/10.36956/rwae.v6i3.1880

