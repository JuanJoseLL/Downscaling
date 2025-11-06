# Sistema de Downscaling Climático - Diagrama de Secuencia

## Flujo Completo del Sistema

```mermaid
sequenceDiagram
    participant CHIRPS as CHIRPS<br/>(Precipitación 5.5km)
    participant C3S as Copernicus C3S<br/>(Temp/Humedad 11km)
    participant MODIS as MODIS GEE<br/>(NDVI/NDWI 500m)
    participant MSWEP as MSWEP<br/>(Precipitación 10km)
    participant DEM as DEM<br/>(Elevación 1km)
    participant Stage1 as 01_Preprocesamiento
    participant Stage2 as 02_Feature_Engineering
    participant Stations as Estaciones_Terreno
    participant Stage3 as 03_Modelado
    participant Stage4 as 04_Predicción_Espacial
    participant Output as Mapas_1km

    Note over CHIRPS,MSWEP: ETAPA 1: PREPROCESAMIENTO TEMPORAL

    CHIRPS->>Stage1: GeoTIFF diarios (2013-2025)
    C3S->>Stage1: NetCDF diarios (tmax, tmin, humidity, wind)
    MODIS->>Stage1: GeoTIFF diarios (NDVI, NDWI)
    MSWEP->>Stage1: NetCDF diarios (precipitación)

    Stage1->>Stage1: Convertir días → Semanas Epidemiológicas
    Stage1->>Stage1: Control de calidad (valores fuera de rango)
    Stage1->>Stage1: Interpolación espacial de NaNs

    Stage1-->>Stage1: chirps_2013-22_2025-05.nc<br/>(610 semanas)
    Stage1-->>Stage1: c3s_2013-22_2025-05.nc<br/>(4 variables × 610 semanas)
    Stage1-->>Stage1: gee_2013-22_2025-05.nc<br/>(NDVI/NDWI × 608 semanas)
    Stage1-->>Stage1: mswep_2013-22_2025-05.nc<br/>(610 semanas)

    Note over DEM,Stage2: ETAPA 2: INGENIERÍA DE VARIABLES

    DEM->>Stage2: DEM.tif (EPSG:4326 → EPSG:3116)
    Stage1->>Stage2: Datos semanales agregados

    Stage2->>Stage2: Calcular variables topográficas:<br/>• Slope (pendiente)<br/>• Aspect (orientación)<br/>• TPI (posición topográfica)

    Stage2->>Stage2: Calcular distancias:<br/>• dist_cumbre (TPI > 100)<br/>• dist_valle (TPI < -100)

    Stage2->>Stage2: Calcular rugosidad:<br/>• Rango elevación ventana 3×3

    Stage2->>Stage2: DOWNSCALING FÍSICO (SAGA):<br/>tmax_11km + DEM → tmax_saga_1km<br/>Lapse Rate: -6.5°C/km

    Stage2->>Stage2: Reproyectar NDVI/NDWI:<br/>MODIS 500m → 1km (EPSG:4326)

    Stage2->>Stage2: Variables temporales:<br/>• week_sin = sin(2π×week/52)<br/>• week_cos = cos(2π×week/52)

    Stations->>Stage2: Coordenadas estaciones (5 sitios)<br/>120,658 observaciones diarias

    Stage2->>Stage2: Extracción por coordenadas:<br/>extract_from_da_exact_week()

    Stage2-->>Stage2: tmax_features.parquet<br/>(1,420 rows × 17 cols)
    Stage2-->>Stage2: tmin_features.parquet<br/>(1,420 rows × 17 cols)
    Stage2-->>Stage2: precip_features.parquet<br/>(9,823 rows × 20 cols)

    Note over Stage2,Stage3: ETAPA 3: MODELADO Y OPTIMIZACIÓN

    Stage2->>Stage3: Features extraídas (.parquet)

    Stage3->>Stage3: BASELINE - Regresión Lineal:<br/>Grid search (5 vars, grado 1-2)<br/>CV temporal (bloques 5-26 semanas)

    Stage3-->>Stage3: Mejor combinación Tmax:<br/>['tmax_saga', 'tmin_saga', 'aspect', 'valle', 'ndvi']<br/>RMSE: ~1.0°C | Pearson: 0.91

    Stage3->>Stage3: OPTIMIZACIÓN XGBOOST:<br/>Optuna Bayesian (100 trials)<br/>Storage: SQLite persistent

    Stage3->>Stage3: Temperatura:<br/>• Sample weighting = 1 + ((T-μ)/σ)²<br/>• CV temporal: bloques 5 semanas<br/>• n_estimators: 259, max_depth: 3

    Stage3->>Stage3: Precipitación:<br/>• Objective: reg:tweedie (p=1.87)<br/>• CV espaciotemporal (LOSO + 26 semanas)<br/>• Paralelización: 56 workers<br/>• n_estimators: 517, max_depth: 5

    Stage3->>Stage3: SHAP Values (interpretabilidad)

    Stage3-->>Stage3: xgb_tmax.joblib (242 KB)<br/>7 features finales
    Stage3-->>Stage3: xgb_tmin.joblib (74 KB)<br/>7 features finales
    Stage3-->>Stage3: xgb_precip.joblib (1.1 MB)<br/>8 features finales

    Note over Stage3,Output: ETAPA 4: PREDICCIÓN ESPACIAL

    Stage3->>Stage4: Modelos entrenados (.joblib)
    Stage1->>Stage4: Datos NetCDF semanales
    DEM->>Stage4: Variables topográficas

    Stage4->>Stage4: Cargar shapefile región:<br/>• Cali (urbano)<br/>• Valle (departamento)<br/>• Colombia (país)

    Stage4->>Stage4: Reproyección a malla común:<br/>rio.reproject_match() → 1km EPSG:4326

    Stage4->>Stage4: Clipping geográfico:<br/>Buffer 0.004° en bordes

    Stage4->>Stage4: Preparar grilla completa:<br/>xr.Dataset → DataFrame → dropna()

    Stage4->>Stage4: Calcular variables derivadas:<br/>• week_sin/cos<br/>• mswep_dem (precip)<br/>• ndvi_rugosity (precip)

    Stage4->>Stage4: predict_from_dataset():<br/>Aplicar modelo pixel-a-pixel

    Stage4-->>Output: tmax_predicted.nc<br/>(610 weeks × ~100×150 pixels)
    Stage4-->>Output: tmin_predicted.nc<br/>(610 weeks × ~100×150 pixels)
    Stage4-->>Output: precip_predicted.nc<br/>(610 weeks × ~100×150 pixels)

    Output->>Output: Visualización con Cartopy:<br/>• Mapas temporales<br/>• Comparación AgERA5 vs SAGA vs XGBoost

    Note over CHIRPS,Output: RESULTADOS: Mejora 5°C RMSE (temp), 10mm RMSE (precip)
```

---

## Diagrama de Flujo de Datos (Simplificado)

```mermaid
flowchart TD
    subgraph Input["FUENTES DE DATOS SATELITALES"]
        A1[CHIRPS<br/>5.5km diarios]
        A2[Copernicus C3S<br/>11km diarios]
        A3[MODIS GEE<br/>500m diarios]
        A4[MSWEP<br/>10km diarios]
        A5[DEM<br/>1km estático]
    end

    subgraph Stage1["ETAPA 1: PREPROCESAMIENTO"]
        B1[Agregación Temporal<br/>→ Semanas Epidemiológicas]
        B2[Control de Calidad<br/>Filtros + Interpolación]
        B3[NetCDF Semanales<br/>610 semanas<br/>2013-2025]
    end

    subgraph Stage2["ETAPA 2: FEATURE ENGINEERING"]
        C1[Variables Topográficas<br/>Slope, Aspect, TPI,<br/>Rugosity, Distancias]
        C2[Downscaling Físico<br/>SAGA Lapse Rate<br/>-6.5°C/km]
        C3[Reproyección NDVI/NDWI<br/>500m → 1km]
        C4[Variables Temporales<br/>sin/cos semanales]
        C5[Extracción en Estaciones<br/>5 sitios × 610 semanas]
        C6[Features .parquet<br/>1,420-9,823 filas]
    end

    subgraph Stage3["ETAPA 3: MODELADO ML"]
        D1[Baseline Lineal<br/>Grid Search<br/>Polinomial Grado 2]
        D2[XGBoost Optimización<br/>Optuna 100 trials<br/>Bayesian Search]
        D3[Validación CV<br/>Temporal/Espaciotemporal<br/>LOSO + Bloques]
        D4[Modelos .joblib<br/>242KB - 1.1MB]
    end

    subgraph Stage4["ETAPA 4: PREDICCIÓN ESPACIAL"]
        E1[Grilla Completa 1km<br/>Malla Alineada]
        E2[Clipping Geográfico<br/>Shapefiles Regionales]
        E3[Predicción Píxel-a-Píxel<br/>610 semanas]
        E4[Mapas NetCDF 4D<br/>epi_week × y × x]
    end

    subgraph Output["SALIDAS"]
        F1[Mapas 1km<br/>Tmax/Tmin/Precip]
        F2[Visualizaciones<br/>Cartopy + Matplotlib]
        F3[Métricas<br/>RMSE: ~1°C temp<br/>~26mm precip]
    end

    A1 & A2 & A3 & A4 --> B1
    B1 --> B2 --> B3

    B3 --> C1 & C2 & C3 & C4
    A5 --> C1 & C2
    C1 & C2 & C3 & C4 --> C5
    C5 --> C6

    C6 --> D1 --> D2
    D2 --> D3 --> D4

    D4 --> E1
    B3 --> E1
    A5 --> E1
    E1 --> E2 --> E3 --> E4

    E4 --> F1 --> F2 --> F3

    style Input fill:#e1f5ff
    style Stage1 fill:#fff4e1
    style Stage2 fill:#e8f5e9
    style Stage3 fill:#f3e5f5
    style Stage4 fill:#ffe0e0
    style Output fill:#fff9c4
```

---

## Arquitectura de Modelos XGBoost

```mermaid
graph TD
    subgraph Temperatura["MODELO TEMPERATURA (Tmax/Tmin)"]
        T1[Features: 7 variables<br/>aspect, ndvi, slope,<br/>tmax_saga, tmin_saga,<br/>valle, week_sin]
        T2[XGBoost Regressor<br/>n_estimators: 259<br/>max_depth: 3<br/>learning_rate: 0.034]
        T3[Sample Weighting<br/>w = 1 + ((T-μ)/σ)²]
        T4[CV Temporal<br/>Bloques 5 semanas]
        T5[Output: Tmax/Tmin 1km<br/>RMSE: ~1°C<br/>Pearson: 0.91]

        T1 --> T2
        T3 --> T2
        T2 --> T4
        T4 --> T5
    end

    subgraph Precipitacion["MODELO PRECIPITACIÓN"]
        P1[Features: 8 variables<br/>Latitud, mswep_1km,<br/>mswep_dem, ndvi, ndwi,<br/>tmax_saga, tmin_saga, week_sin]
        P2[XGBoost Tweedie<br/>n_estimators: 517<br/>max_depth: 5<br/>learning_rate: 0.048<br/>tweedie_variance: 1.87]
        P3[CV Espaciotemporal<br/>LOSO + Bloques 26 semanas<br/>Paralelo: 56 workers]
        P4[Output: Precipitación 1km<br/>RMSE: ~26mm<br/>Pearson: 0.77]

        P1 --> P2
        P2 --> P3
        P3 --> P4
    end

    style Temperatura fill:#ffebee
    style Precipitacion fill:#e3f2fd
```

---

## Cronología del Pipeline

```mermaid
gantt
    title Pipeline de Downscaling Climático (2013-2025)
    dateFormat YYYY-MM-DD
    section Adquisición Datos
    CHIRPS diarios          :2013-06-01, 2025-01-31
    C3S/AgERA5 diarios      :2013-06-01, 2025-01-31
    MODIS diarios           :2013-06-01, 2025-01-31
    MSWEP diarios           :2013-06-01, 2025-01-31

    section Procesamiento
    Notebook 01 Preprocesamiento :milestone, 2024-01-01, 0d
    Agregación semanal      :2024-01-01, 7d

    section Feature Engineering
    Notebook 02 Features    :milestone, 2024-01-08, 0d
    Variables topográficas  :2024-01-08, 5d
    SAGA downscaling        :2024-01-13, 3d
    Extracción estaciones   :2024-01-16, 2d

    section Modelado
    Notebook 03 Modeling    :milestone, 2024-01-18, 0d
    Baseline lineal         :2024-01-18, 2d
    Optimización XGBoost    :2024-01-20, 5d
    Validación SHAP         :2024-01-25, 1d

    section Predicción
    Notebook 04 Spatial     :milestone, 2024-01-26, 0d
    Predicción espacial     :2024-01-26, 3d
    Visualización mapas     :2024-01-29, 1d
```

---

## Métricas de Validación

```mermaid
graph LR
    subgraph Validación["ESTRATEGIA DE VALIDACIÓN"]
        V1[Temperatura<br/>Leave-5-Weeks-Out<br/>Temporal CV]
        V2[Precipitación<br/>LOSO Espacial +<br/>26-Weeks Temporal]

        V1 --> M1[Métricas<br/>RMSE, Pearson r,<br/>Bias, Var Ratio]
        V2 --> M1
    end

    subgraph Resultados["RESULTADOS POR ESTACIÓN"]
        R1[Cañaveralejo<br/>SAGA: RMSE 6.3°C<br/>XGBoost: RMSE 1.0°C<br/>Mejora: 84%]
        R2[Villanueva<br/>MSWEP: RMSE 30.8mm<br/>XGBoost: RMSE 16.1mm<br/>Mejora: 48%]
    end

    M1 --> R1 & R2

    style Validación fill:#e8f5e9
    style Resultados fill:#fff9c4
```

