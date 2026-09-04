# Predicción espacio-temporal adaptativa de casos de malaria a nivel distrital en el Perú

### Documento maestro del proyecto — versión de trabajo

---

## 0. Resumen ejecutivo (para lectura rápida del equipo)

Vamos a construir y comparar una escalera de modelos —desde estadísticos simples hasta un modelo espacio-temporal adaptativo propio— para predecir el número semanal de casos de malaria por distrito en el Perú, con horizontes de predicción de 1, 4 y 12 semanas. Usamos datos de vigilancia epidemiológica del MINSA (2009–2024, 597,141 casos individuales), agregados a nivel **UBIGEO × semana epidemiológica**. El problema central que atacamos es que los modelos convencionales asumen relaciones estáticas y distribuciones simples (ej. normal, Poisson), pero los datos reales muestran **sobredispersión extrema** (var/media ≈ 72), **73% de observaciones en cero** incluso en distritos endémicos, **fuerte dependencia espacial** (86.6% de los casos concentrados en Loreto) y **cambios de régimen no climáticos** (intervenciones sanitarias 2006–2010, pandemia COVID-19). Nuestra contribución es un modelo que integra estos tres elementos —naturaleza de conteo, dependencia espacio-temporal, y adaptación a cambios de régimen— en una sola arquitectura, evaluada mediante un *ablation study* contra una escalera de baselines.

---

## 1. Planteamiento del problema

### 1.1 Contexto

La malaria sigue siendo un problema de salud pública activo en el Perú, concentrado principalmente en la Amazonía (Loreto) pero con presencia endémica en al menos otros 20 departamentos. Los sistemas de alerta temprana actuales dependen en gran medida de factores climáticos locales de corto plazo o de índices climáticos globales estáticos (como ENSO), y rara vez modelan explícitamente:

- La naturaleza de **conteo con exceso de ceros** de los datos epidemiológicos a nivel distrital.
- La **dependencia espacial** entre distritos vecinos (migración, movilidad, corredores fluviales).
- Los **cambios de patrón epidemiológico** causados por factores no climáticos (campañas de control vectorial, disrupciones como la pandemia COVID-19).

### 1.2 Problema

La variación espacial y temporal de los casos de malaria, junto con cambios en sus patrones epidemiológicos a lo largo del tiempo, dificulta la generación de predicciones precisas y robustas mediante modelos estáticos convencionales.

### 1.3 Pregunta de investigación

¿En qué medida un modelo espacio-temporal adaptativo puede mejorar la predicción multihorizonte de casos de malaria a nivel distrital en el Perú frente a modelos convencionales, bajo cambios en los patrones epidemiológicos?

### 1.4 Hipótesis

Un modelo espacio-temporal adaptativo presenta mejor desempeño predictivo y mayor robustez frente a cambios epidemiológicos que los modelos convencionales (estadísticos de conteo simples, ML tabular no espacial, y modelos espacio-temporales estáticos).

### 1.5 Objetivo general

Desarrollar y evaluar un modelo espacio-temporal adaptativo para la predicción multihorizonte de casos de malaria a nivel distrital en el Perú, considerando la naturaleza de conteo, las dependencias espaciales y temporales, y los cambios en los patrones epidemiológicos.

### 1.6 Objetivos específicos

1. Construir un panel limpio y completo de casos de malaria a nivel UBIGEO × semana epidemiológica (2009–2024), documentando y resolviendo los problemas de calidad de datos identificados.
2. Establecer una escalera de modelos de referencia (*baselines*) que respeten progresivamente más propiedades de los datos: conteo, sobredispersión, exceso de ceros, no linealidad y dependencia espacial.
3. Diseñar y entrenar una arquitectura espacio-temporal adaptativa que modele explícitamente la naturaleza de conteo de los casos.
4. Evaluar mediante un *ablation study* cuánto aporta cada componente de la arquitectura final (espacialidad, adaptación a cambios de régimen, cabeza de conteo) frente a quitarlo.
5. Comparar el desempeño de todos los modelos en múltiples horizontes de predicción (1, 4 y 12 semanas) usando métricas apropiadas para datos de conteo.

### 1.7 Contribución esperada

Integración, en una sola arquitectura evaluable, de: (a) modelado explícito de la naturaleza de conteo (sobredispersión + exceso de ceros), (b) dependencia espacio-temporal vía grafos, y (c) adaptación a cambios de régimen epidemiológico — evaluada de forma rigurosa mediante *ablations* que aíslan el aporte de cada componente, no solo un desempeño agregado.

---

## 2. Alcance y delimitación

Estas delimitaciones fueron necesarias tras revisar el dataset real (ver sección 3) y deben citarse explícitamente en el protocolo como decisiones metodológicas justificadas, no como debilidades ocultas.

| Dimensión | Delimitación adoptada | Justificación |
|---|---|---|
| **Geográfica** | Universo de estudio principal: distritos con ≥20 casos acumulados en 2009–2024 (161 de 1,874 distritos del país; retiene 99.83% de los casos totales). **Decisión confirmada, no se reabre**: es el balance correcto entre retener casi todos los casos reales y mantener un grafo del ST-GNN de tamaño manejable (ni tan chico que se pierda estructura espacial, ni tan grande que se vuelva lento/inestable de entrenar en CPU). No se introduce como variable adicional a decidir en el pipeline; si al final del *ablation study* sobra tiempo, se prueba como análisis de sensibilidad, pero no bloquea el desarrollo. El resto del país queda fuera del entrenamiento principal, disponible solo para pruebas de generalización si el tiempo lo permite. | Evita inflar artificialmente el problema de ceros con distritos sin transmisión histórica; sigue el mismo criterio usado por estudios de referencia (Pan et al., 2026, se limitan a Loreto). |
| **Por especie** | Variable objetivo principal: `casos_total` (P. vivax + P. falciparum combinados). Separación por especie queda como análisis de sensibilidad/extensión. | Mantiene mayor señal y menos dispersión que separar especies desde el inicio; es comparable con el enfoque de la literatura de referencia. |
| **Temporal** | 2009–2024 (16 años), semana epidemiológica MINSA (1–53), con la semana 53 colapsada en la semana 52 (ver sección 3.4). | Cobertura completa disponible en el dataset oficial. |
| **Unidad de análisis** | UBIGEO (distrito) × semana epidemiológica. | Nivel de granularidad que exige el diseño del proyecto (predicción "distrital"). |
| **Horizontes de predicción** | 1, 4 y 12 semanas (corto, mediano, largo/long-lead). | Permite comparar desempeño en el rango donde los modelos simples suelen fallar (>4 semanas), replicando la lógica de "long-lead prediction" de la literatura de referencia. |
| **Alcance causal** | El proyecto es de **predicción (forecasting)**, no de inferencia causal de determinantes de malaria. | Evita sobre-interpretar pesos/coeficientes de los modelos como relaciones causales. |

---

## 3. Datos

### 3.1 Fuente

Dataset de vigilancia epidemiológica de malaria, MINSA Perú, datos abiertos, periodo 2009–2024. Formato original: línea de casos individuales (una fila = un caso reportado).

### 3.2 Estructura original (línea de casos)

| Columna | Descripción |
|---|---|
| `departamento`, `provincia`, `distrito`, `localidad` | Ubicación geográfica del caso |
| `enfermedad` | `MALARIA POR P. VIVAX` o `MALARIA P. FALCIPARUM` |
| `ano`, `semana` | Año y semana epidemiológica de reporte |
| `diagnostic` | Código CIE-10 (B51 = vivax, B50 = falciparum) |
| `diresa` | Código de dirección regional de salud |
| `ubigeo` | Código distrital (6 dígitos) — clave para la agregación espacial |
| `localcod` | Código de centro poblado (granularidad más fina, no usada en el análisis principal) |
| `edad`, `tipo_edad`, `sexo` | Variables demográficas del caso |

### 3.3 Cifras descriptivas clave (ya validadas en notebook 01)

- **597,141 casos** individuales, 2009–2024.
- **490,097 (82.1%) P. vivax**, **107,044 (17.9%) P. falciparum**.
- **86.6% de los casos concentrados en Loreto**; el resto disperso en 21 departamentos.
- **396 UBIGEOs distintos** reportaron al menos un caso alguna vez, de ~1,874 distritos del país.
- Tras filtrar al universo endémico (≥20 casos acumulados → 161 distritos) y expandir a la grilla completa UBIGEO × semana: **134,435 observaciones**, de las cuales **73.02% son cero**.
- **Sobredispersión extrema**: media = 4.43 casos/semana/distrito, varianza = 320.34 → ratio var/media = **72.24**.

### 3.4 Problemas de calidad detectados y resueltos (ver notebook 01)

| Problema | Magnitud | Tratamiento aplicado |
|---|---|---|
| Edades implausibles (hasta ~90 millones, probable DNI mal ingresado) | 44 registros | Invalidadas (NaN), sin descartar el caso |
| Unidades de edad mixtas (años/meses/días) | 7,658 + 109 registros | Homogenizadas a años decimales |
| Filas duplicadas exactas | 57,458 (9.62%) | Marcadas con bandera `es_duplicado_exacto`; **decisión confirmada: se deduplican** (parámetro `DEDUPLICAR = True`) |
| Valores faltantes en `localidad`/`localcod` | 18,593 / 5,502 | No crítico (no se usan a nivel distrital) |
| Semana 53 (solo existe en años de 53 semanas ISO) | — | **Decisión confirmada: se colapsa con la semana 52**. Es el estándar en vigilancia epidemiológica (MINSA y la mayoría de sistemas de vigilancia lo hacen así): la semana 53 solo aparece en años ISO con 53 semanas, tiene muy pocas observaciones, y dejarla aparte crea una categoría rara con muestra mínima que solo mete ruido al modelar la estacionalidad (semana del año como variable cíclica de 52 categorías, no 53). Colapsarla mantiene el ciclo estacional limpio y consistente todos los años. |

### 3.5 Variable objetivo

`casos_total`: número de casos de malaria (P. vivax + P. falciparum) reportados por distrito (UBIGEO) en una semana epidemiológica dada. Variable de **conteo, no negativa, con sobredispersión y exceso de ceros**.

### 3.6 Variables predictoras / covariables (a incorporar progresivamente)

| Tipo | Variable | Notebook donde se incorpora |
|---|---|---|
| **Autoregresiva** | Casos en semanas t-1, t-4, t-12 (lags) | 02 en adelante |
| **Estacional** | Semana del año, mes, indicador de temporada de lluvias (~enero-mayo) | 02 en adelante |
| **Espacial (estructural)** | Matriz de adyacencia/grafo entre distritos (contigüidad, distancia, o proxy de movilidad) | 05–08 |
| **Climática (opcional/extensión)** | Temperatura, precipitación local; posible índice SST/ENSO como covariable exógena de largo plazo | Evaluar en extensión, no bloqueante para el pipeline principal |
| **Categórica** | Departamento/UBIGEO (para modelos que lo requieran como dummy o embedding) | 03 en adelante |
| **De régimen (para el componente adaptativo)** | Indicador de intervención sanitaria conocida (si se reconstruye), o señal de drift detectada de forma no supervisada | 07–08 |

---

## 4. Marco de referencia

Referencia metodológica principal: Pan, M. et al. (2026). *A Machine Learning-Based Dynamic SST Index for Long-Lead Malaria Prediction in the Peruvian Amazon*. GeoHealth.

Elementos que tomamos de esa referencia:
- Uso de **GLM Binomial Negativo** para modelar conteos de malaria con sobredispersión.
- Validación mediante **bootstrap/cross-validation con splits temporales**, evaluando correlación y RMSE.
- Evaluación explícita en **múltiples horizontes (lead times)**, no solo un horizonte fijo.
- Reconocimiento explícito de que **factores no climáticos** (intervenciones, COVID-19) degradan el desempeño predictivo — motivó directamente nuestro componente adaptativo.

Lo que extendemos más allá de esa referencia:
- Trabajamos a **nivel distrital** (UBIGEO), no agregado regional.
- Incorporamos **dependencia espacial explícita** entre distritos (ellos no la modelan).
- Incorporamos un **mecanismo de adaptación a cambios de régimen**, en vez de un único modelo estático para todo el periodo.
- Comparamos contra una **escalera completa de modelos**, no solo dos (ellos comparan su índice dinámico vs. ENSO únicamente).

---

## 5. Metodología: hoja de ruta paso a paso

### Notebook 01 — Carga, limpieza y baseline (✅ completado)
- Carga del dataset de línea de casos.
- Limpieza (edades, duplicados, unidades).
- Definición del universo endémico.
- Agregación a panel UBIGEO × semana, con grilla completa (ceros explícitos).
- Diagnóstico de sobredispersión y exceso de ceros.
- Función de partición *walk-forward*.
- GLM Binomial Negativo de prueba (un solo distrito, para validar el pipeline).
- **Salida**: `panel_ubigeo_semana.parquet` — insumo de todos los notebooks siguientes.

### Notebook 02 — Baselines estadísticos (todos los distritos)
- Seasonal Naive (predicción = mismo valor semana equivalente año anterior, o promedio histórico de esa semana).
- GLM Poisson (para mostrar explícitamente su falla por sobredispersión).
- GLM Binomial Negativa, ahora corrido sobre los 161 distritos (no solo el de ejemplo).
- ZINB / Hurdle-NB, para manejar el exceso de ceros de forma explícita.
- Evaluación con walk-forward en horizontes 1/4/12 semanas.
- **Salida esperada**: tabla comparativa RMSE/correlación/log-score por modelo y horizonte; primeras figuras (ver sección 7).

### Notebook 03 — Benchmark ML no espacial
- Ingeniería de *features*: lags múltiples, medias móviles, semana del año, dummies de distrito/departamento.
- XGBoost / LightGBM.
- Comparación contra el mejor modelo del notebook 02.

### Notebook 04 — Benchmark espacio-temporal interpretable
- Modelo `hhh4` (endemic-epidemic framework) o alternativa bayesiana espacial (BYM2-NB).
- Este modelo es clave para el "puente" interpretable entre lo estadístico y lo deep learning.

### Notebook 05 — Construcción del grafo espacial
- Construcción de 2–3 definiciones de grafo entre los 161 distritos endémicos:
  1. Contigüidad geográfica (comparten frontera).
  2. Distancia entre centroides distritales.
  3. Proxy de movilidad (ej. red fluvial/vial, si se logra reconstruir).
- Visualización de cada grafo sobre el mapa del Perú.
- Análisis de sensibilidad: mismo modelo corrido con cada definición de grafo.

### Notebook 06 — Arquitectura espacio-temporal completa (ST-GNN)
- Implementación de la arquitectura base (tipo STGCN o DCRNN), con cabeza de conteo (NB/ZINB) desde el inicio.
- Entrenamiento con los splits walk-forward, horizontes 1/4/12 semanas.

### Notebook 07 — Componente adaptativo
- Incorporación del mecanismo de detección de cambio de régimen (ej. Page-Hinkley o ADWIN sobre residuales, o ventana deslizante con recalibración).
- Entrenamiento y evaluación de la arquitectura adaptativa completa.

### Notebook 08 — Ablation study y comparación final
- Variante sin grafo espacial (≈ LSTM/GRU puro) — aísla el aporte del componente espacial.
- Variante sin mecanismo adaptativo (ST-GNN estático) — aísla el aporte de la adaptación.
- Variante con cabeza MSE en vez de NB/ZINB — aísla el aporte de modelar conteo explícitamente.
- Tabla y gráficos comparativos finales, con todos los modelos (01 al 08) en las mismas condiciones de evaluación.

---

## 6. Modelos a comparar (resumen)

| # | Modelo | Rol en la escalera | Notebook |
|---|---|---|---|
| 1 | Seasonal Naive | Piso absoluto | 02 |
| 2 | GLM Poisson | Referencia que falla por sobredispersión (a propósito) | 02 |
| 3 | GLM Binomial Negativa | Referencia estadística estándar | 02 |
| 4 | ZINB / Hurdle-NB | Maneja exceso de ceros explícitamente | 02 |
| 5 | XGBoost / LightGBM | Benchmark ML no espacial | 03 |
| 6 | `hhh4` / BYM2-NB | Espacio-temporal interpretable | 04 |
| 7 | ST-GNN (STGCN/DCRNN) + cabeza NB/ZINB | Espacio-temporal no lineal | 06 |
| 8 | ST-GNN adaptativo (con detección de drift) | Arquitectura final propuesta | 07 |
| 8a | Ablation: sin grafo | Aísla aporte espacial | 08 |
| 8b | Ablation: sin adaptación | Aísla aporte de adaptación | 08 |
| 8c | Ablation: cabeza MSE | Aísla aporte de modelar conteo | 08 |

---

## 7. Visualizaciones y productos esperados (más allá del entrenamiento)

Cada notebook debe producir, además de las métricas, evidencia visual que soporte la narrativa metodológica del proyecto:

### 7.1 Exploración de datos (notebook 01, ya generado / extender)
- Serie temporal nacional agregada (semanal), con anotaciones de periodos conocidos (intervención 2006–2010 fuera de rango, COVID 2020–2021).
- Series temporales de los distritos con más casos acumulados (ya generado).
- **Mapa coroplético del Perú** con casos acumulados totales por distrito (2009–2024) — visualiza la concentración en Loreto.
- **Mapa coroplético animado o por facetas por año**, para visualizar la evolución espacial en el tiempo.

### 7.2 Estructura espacio-temporal (notebook 02 en adelante)
- **Heatmap de correlación cruzada** entre series de distritos vecinos (matriz distrito × distrito), análogo a los mapas de correlación lead-lag del paper de referencia, pero para dependencia espacial local.
- **Heatmap semana epidemiológica × año** por distrito (tipo calendario), para visualizar estacionalidad y anomalías (similar a un mapa de calor de "carga viral" usado en vigilancia epidemiológica).
- Gráfico de proporción de ceros por distrito (barras ordenadas), para justificar visualmente el uso de ZINB.

### 7.3 Grafo espacial (notebook 05)
- **Mapa del Perú con los nodos (distritos) y aristas (conexiones del grafo)** superpuestas, para cada una de las 2–3 definiciones de grafo probadas.
- Comparación visual lado a lado de las 3 estructuras de grafo.

### 7.4 Desempeño de modelos (todos los notebooks, consolidado en 08)
- Gráfico de **predicho vs. observado** por modelo y horizonte (como el de la Figura 4f del paper de referencia).
- **Curvas de RMSE/correlación por horizonte de predicción** (1 a 12+ semanas), una línea por modelo — el gráfico central de toda la tesis.
- **Boxplots de desempeño en validación walk-forward** (múltiples splits), para mostrar robustez y no solo un punto de desempeño.
- Mapa coroplético de **error de predicción (RMSE) por distrito**, para el mejor modelo — identifica dónde predice bien y dónde no.
- Gráfico de barras del **ablation study**: desempeño del modelo completo vs. cada variante reducida, mismo horizonte.

### 7.5 Interpretabilidad (si aplica al modelo final)
- Visualización de pesos de atención espacial o temporal del ST-GNN (si la arquitectura elegida los expone).
- Visualización de los puntos donde el mecanismo adaptativo detectó cambios de régimen, superpuestos sobre la serie temporal observada (para validar que coincide con periodos conocidos, ej. COVID-19).

---

## 8. Diseño de validación y métricas

- **Esquema de validación**: *walk-forward* (rolling origin), respetando el orden temporal — nunca se entrena con datos futuros al punto de corte.
- **Horizontes evaluados**: 1, 4 y 12 semanas (multihorizonte, evaluado por separado, no promediado a ciegas).
- **Métricas**:
  - **RMSE** — error absoluto, comparable entre modelos.
  - **Correlación de Pearson** — captura de variabilidad interanual.
  - **Log-score / CRPS** — para modelos con salida probabilística (NB, ZINB, hhh4, y la arquitectura final), evalúa calibración, no solo el punto estimado.
  - **Prueba t pareada** (o equivalente no paramétrico) entre modelos, sobre los resultados de múltiples splits walk-forward, para determinar si las diferencias son estadísticamente significativas.

---

## 9. Supuestos y limitaciones

Ver documento complementario **`supuestos_y_limitaciones.md`** (ya generado) para el detalle completo. Resumen de los puntos más críticos a tener presentes durante el desarrollo:

- El exceso de ceros y la sobredispersión son propiedades **confirmadas empíricamente**, no supuestas — todo modelo debe rendir cuentas de esto.
- Los duplicados exactos (9.62%) ya tienen decisión documentada: se deduplican (ver sección 3.4).
- El desempeño de todos los modelos se espera que se degrade durante 2019–2021 (COVID-19) — esto **no es una falla del modelo**, es el fenómeno que el componente adaptativo debe, idealmente, manejar mejor que los baselines.
- El estudio es de predicción, no de inferencia causal.

---

## 10. Stack tecnológico propuesto

| Componente | Herramienta |
|---|---|
| Manipulación de datos | `pandas`, `numpy`, `pyarrow` |
| Modelos estadísticos de conteo | `statsmodels` (Poisson, NB), `pscl` o implementación propia para ZINB, `surveillance` (R, para `hhh4`) vía `rpy2` si se integra con Python |
| ML tabular | `xgboost`, `lightgbm`, `scikit-learn` |
| Deep learning / ST-GNN | `PyTorch` + `PyTorch Geometric Temporal` (o `DGL`), según disponibilidad de cómputo (CPU-friendly por defecto, escalable a GPU) |
| Mapas y geoespacial | `geopandas`, `shapely`, límites distritales del INEI (shapefile UBIGEO) |
| Visualización | `matplotlib`, `seaborn`, `plotly` (para gráficos interactivos si se desea) |
| Notebooks | Jupyter (`.ipynb`), uno por etapa de la hoja de ruta (sección 5) |

---

## 11. Estructura de entregables (para el equipo)

```
proyecto_malaria/
├── data/
│   ├── raw/                        # CSV original MINSA (no versionar en git)
│   └── processed/
│       └── panel_ubigeo_semana.parquet
├── notebooks/
│   ├── 01_carga_limpieza_baseline.ipynb        ✅ hecho
│   ├── 02_baselines_estadisticos.ipynb
│   ├── 03_ml_tabular.ipynb
│   ├── 04_hhh4_bym2.ipynb
│   ├── 05_construccion_grafo.ipynb
│   ├── 06_stgnn.ipynb
│   ├── 07_stgnn_adaptativo.ipynb
│   └── 08_ablation_comparacion_final.ipynb
├── figuras/                        # mapas, heatmaps, curvas de desempeño (salida de cada notebook)
├── docs/
│   ├── documento_maestro_proyecto.md   (este documento)
│   └── supuestos_y_limitaciones.md
└── README.md
```

---

## 12. Próximos pasos inmediatos

1. ~~Confirmar decisiones pendientes: deduplicación, tratamiento de semana 53, umbral definitivo de universo endémico.~~ **Confirmado** (ver secciones 2 y 3.4): se deduplica, semana 53 colapsada en semana 52, umbral endémico ≥20 casos acumulados (161 distritos).
2. Construir notebook 02 (baselines estadísticos completos, los 161 distritos).
3. Conseguir/descargar shapefile de límites distritales (INEI) para los mapas coropléticos.
4. Definir el entorno de cómputo final (CPU/GPU) antes del notebook 06, para dimensionar la arquitectura ST-GNN.
