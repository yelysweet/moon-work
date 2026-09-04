# Predicción espacio-temporal adaptativa de casos de malaria a nivel distrital en el Perú

Proyecto que construye y compara una escalera de modelos —desde estadísticos simples hasta un
modelo espacio-temporal adaptativo propio (ST-GNN)— para predecir el número semanal de casos de
malaria por distrito en el Perú, en horizontes de 1, 4 y 12 semanas.

Ver el documento completo del proyecto en [`docs/documento_maestro_proyecto.md`](docs/documento_maestro_proyecto.md).

## Estructura del repositorio

```
proyecto_malaria/
├── data/
│   ├── raw/                        # CSV original MINSA (no versionado, ver abajo)
│   └── processed/
│       └── panel_ubigeo_semana.parquet   # generado por notebook 01 (no versionado)
├── notebooks/
│   ├── 01_carga_limpieza_baseline.ipynb
│   ├── 02_baselines_estadisticos.ipynb
│   ├── 03_ml_tabular.ipynb
│   ├── 04_hhh4_bym2.ipynb
│   ├── 05_construccion_grafo.ipynb
│   ├── 06_stgnn.ipynb
│   ├── 07_stgnn_adaptativo.ipynb
│   └── 08_ablation_comparacion_final.ipynb
├── figuras/                        # mapas, heatmaps, curvas de desempeño
├── docs/
│   ├── documento_maestro_proyecto.md
│   └── supuestos_y_limitaciones.md
└── README.md
```

## Datos

El dataset crudo (línea de casos, MINSA, 2009–2024) **no está versionado** en este repositorio
por su tamaño y naturaleza de datos de salud. Para reproducir el pipeline:

1. Descarga el dataset de vigilancia epidemiológica de malaria desde el portal de datos abiertos del MINSA.
2. Colócalo en `data/raw/`.
3. Ejecuta `notebooks/01_carga_limpieza_baseline.ipynb`, que genera `data/processed/panel_ubigeo_semana.parquet`.

## Decisiones metodológicas confirmadas

- **Deduplicación**: se deduplican las filas exactas antes de agregar.
- **Semana 53**: se colapsa con la semana 52 (estándar de vigilancia epidemiológica).
- **Universo endémico**: distritos con ≥20 casos acumulados en 2009–2024 (161 distritos, 99.83% de cobertura).

Detalle completo y justificación en `docs/documento_maestro_proyecto.md`, sección 2 y 3.4.

## Instalación

```bash
pip install -r requirements.txt
```

## Cómo ejecutar

Los notebooks están numerados en el orden en que deben ejecutarse (01 → 08). Cada uno documenta
su salida esperada al inicio.
