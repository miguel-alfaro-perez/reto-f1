# Reto F1: probabilidades de podio, top 10 y posición final

Proyecto para el reto de la comunidad: predecir, después de la clasificación y antes de la carrera, el resultado de cada piloto en los Grandes Premios de 2024.

- **Parte 1.** `p_podio = P(positionOrder ≤ 3)` y `p_top10 = P(positionOrder ≤ 10)`, con `p_podio ≤ p_top10`. Métrica: media de las dos log loss.
- **Parte 2.** `posicion_pred`, convertida en orden dentro de cada carrera. Métrica: Spearman medio por carrera (desempate: MAE).

## Estructura

```
.
├── reto_f1.ipynb      # notebook único, se ejecuta de arriba abajo (SEED = 2502)
├── predicciones_2024.csv  # entrega: una fila por resultado de 2024
├── docs/bitacora.md   # registro de decisiones y resultados de cada paso
├── docs/diccionario_datos.md  # qué significa cada columna y variable
├── requirements.txt
└── *.csv              # datos de Kaggle (no se suben al repositorio)
```

## Datos

Dataset de Kaggle [Formula 1 World Championship (1950–2024)](https://www.kaggle.com/datasets/rohanrao/formula-1-world-championship-1950-2020). Descarga los CSV y colócalos en la misma carpeta que el notebook.

## Cómo ejecutarlo

```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute reto_f1.ipynb
```

## Estado

| Paso | Estado |
|---|---|
| 1. Carga, uniones y variables prohibidas | hecho |
| 2. EDA | hecho |
| 3. Ingeniería de variables | hecho |
| 4. Validación temporal y referencia de parrilla | hecho |
| 5. Parte 1: probabilidades | hecho |
| 6. Parte 2: posición | hecho |
| 7. Análisis de errores | hecho |
| 8. Exportación de la entrega | hecho |

## Resultados

Validación (entrenando con 2014-2022 y midiendo en 2023):

| Modelo | Parte 1: log loss media (↓) | Parte 2: Spearman medio (↑) |
|---|---|---|
| Solo parrilla | 0,391 | 0,597 |
| Modelo final | 0,339 | 0,668 |

Test 2024 (entrenando con 2014-2023; calculado después de exportar, no se usó para ninguna decisión):

| Modelo | Parte 1: log loss media (↓) | Parte 2: Spearman medio (↑) |
|---|---|---|
| Solo parrilla | 0,337 | 0,729 |
| Modelo final | 0,310 | 0,755 |

- **Parte 1:** media de una regresión logística y un LightGBM, con las probabilidades de cada carrera reescaladas para sumar 3 (podio) y 10 (top 10).
- **Parte 2:** suma del orden que dan una regresión Ridge y un LGBMRanker (lambdarank).

El detalle de cada paso está en `docs/bitacora.md`.
