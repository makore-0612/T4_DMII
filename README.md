<img src="https://mmss.iimas.unam.mx/dmmss/wp-content/uploads/2026/02/cropped-LogoUNAM_IIMAS_Negro50-scaled-2.png" width="200"/>

### Datos Masivos II
# Práctica 4: Sistemas de recomendación con el Netflix Prize
> Equipo Tres: Castrillo Cruz Karen Arlet, Ramos González Nadia, Zamora Antiga Ángel Javier.

---

Este repositorio construye y compara sistemas de recomendación sobre los ~100 millones de ratings del
[Netflix Prize (Kaggle)](https://www.kaggle.com/datasets/netflix-inc/netflix-prize-data): 480,189 usuarios,
17,770 películas, ratings de 1 a 5 entre 1999 y 2005. Usa los métodos del curso: filtrado colaborativo, SVD, NMF,
PCA, t-SNE, A-Priori, PCY, Multistage, Multihash, SON, Toivonen, FP-Growth, PrefixSpan, PageRank por el método de
potencias, grafos y recomendación basada en contenido. Todos se evalúan con el conjunto oficial `probe.txt`.

| Notebook | Contenido | Métodos | Escala |
|---|---|---|---|
| `01_carga_eda.ipynb` | Conversión a Parquet y CSR, exploración, submuestra S | — | completo |
| `02_baseline_vecindarios.ipynb` | Protocolo de evaluación, baseline y vecindarios | media + sesgos (ALS), kNN ítem-ítem coseno / Pearson | completo / S |
| `03_factorizacion.ipynb` | Factorización e interpretación de factores | SVD truncada, PureSVD, SVD con sesgos por SGD, NMF con máscara, PCA, t-SNE | completo / S |
| `04_reglas_secuencias.ipynb` | Itemsets frecuentes, reglas y secuencias | A-Priori, PCY, Multistage, Multihash, SON, Toivonen, FP-Growth, PrefixSpan | S (20,000 usuarios) |
| `05_grafos_contenido.ipynb` | Grafos y contenido | PageRank global y personalizado (método de potencias), paseo P³ bipartito, TF-IDF + año, factores latentes | S |
| `06_comparacion.ipynb` | Tabla final, ensamble y ejemplos | ensamble ridge | S |

```text
T4_DMII/
├── notebooks/
│   ├── 01_carga_eda.ipynb              # Parquet por bloques, CSR, EDA, probe, submuestra S
│   ├── 02_baseline_vecindarios.ipynb   # validación tipo probe, baseline, kNN ítem-ítem (RMSE y top-N)
│   ├── 03_factorizacion.ipynb          # SVD truncada, PureSVD, SVD-SGD, NMF, PCA / t-SNE, interpretación de factores
│   ├── 04_reglas_secuencias.ipynb      # itemsets frecuentes (7 algoritmos), reglas, PrefixSpan y siguiente película
│   ├── 05_grafos_contenido.ipynb       # grafo de películas, PageRank, PPR, P³ bipartito, contenido e híbrido
│   └── 06_comparacion.ipynb            # tabla comparativa, ensamble, ejemplos cualitativos, conclusiones
├── requirements.txt
├── README.md
└── data/                               # no incluida: ver "Cómo replicarlo"
```

## Resultados principales

| Modelo | RMSE probe completo | RMSE probe S | NDCG@10 (S) |
|---|---|---|---|
| Baseline (media + sesgos, 99 M) | 0.9822 | 0.9753 | — |
| kNN ítem-ítem (Pearson + baseline) | — | 0.9312 | 0.0109 |
| **SVD con sesgos por SGD (99 M)** | **0.9130** | **0.9043** | 0.0149 |
| Cinematch (Netflix) | 0.9474 | — | — |
| Popularidad | — | — | 0.0146 |
| kNN, suma de similitudes | — | — | 0.0490 |
| PureSVD ($k = 100$) | — | — | 0.0554 |
| **PageRank personalizado, especificidad** | — | — | **0.0573** |

El SVD-SGD, implementado por nosotros, es **3.6 % mejor que Cinematch** en el probe. Los mejores recomendadores
top-N son los que puntúan por evidencia acumulada y corrigen la popularidad, no los de menor RMSE. La tabla completa,
con 20 modelos, está en `06_comparacion.ipynb`.

## Cómo replicarlo

1. **Datos.** Con la [API de Kaggle](https://www.kaggle.com/docs/api) autenticada, desde la raíz del repositorio:

   ```bash
   kaggle datasets download -d netflix-inc/netflix-prize-data -p data/ --unzip
   ```

   Quedan en `data/` los cuatro `combined_data_*.txt`, `probe.txt`, `qualifying.txt` y `movie_titles.csv` (~2.1 GB).
2. **Dependencias.** `pip install -r requirements.txt`. Los notebooks se ejecutaron con Python 3.14.7 en Windows, con
   12 GB de RAM (~4 GB libres en la práctica).
3. **Orden de ejecución**, desde `notebooks/` (cada uno usa las salidas de los anteriores en `data/processed/`):

   | Notebook | Lee | Genera en `data/processed/` | Tiempo aprox. | Pico de RAM |
   |---|---|---|---|---|
   | `01` | `data/*.txt`, `movie_titles.csv` | `ratings.parquet`, `train_*.npy`, `probe_*.npy`, `R_train.npz`, `S_*` | 8 min (2 min si ya existe el Parquet) | 3.1 GB |
   | `02` | `R_train.npz`, `S_R_train.npz`, probe | `modelos/baseline_*.npz`, `modelos/knn_*`, `resultados/`, `predicciones/` | 15 min | 1.5 GB |
   | `03` | lo anterior | `modelos/svd_*`, `modelos/nmf_S.npz`, `modelos/puresvd_V_S.npy` | 80–90 min (SGD con 99 M: ~30 min) | 2.2 GB |
   | `04` | `S_R_train.npz`, `S_dias.npy` | `modelos/reglas_*.npz` | 10 min | 1.3 GB |
   | `05` | S, `modelos/svd_sgd_S.npz` | `modelos/grafo_peliculas_S.npz`, `modelos/pagerank_S.npz`, `modelos/contenido_S.npz` | 10 min | 1.6 GB |
   | `06` | `resultados/`, `predicciones/`, `modelos/` | `resultados/Ensamble ridge (S).json` | 2 min | < 1 GB |

   Los resultados son reproducibles: todas las partes aleatorias (submuestra, SGD, NMF, `svds`, t-SNE, muestras de
   Toivonen y de la evaluación) usan la semilla 42.

## Decisiones principales

- **Carga.** Los `.txt` se leen por bloques de 5 millones de líneas y nunca se cargan los 100 millones de ratings
  como `DataFrame`. Se usan arreglos `int32`/`int16`/`int8` (9 bytes por rating) y una CSR `float32` armada con un
  ordenamiento por conteo en `numba`.
- **Prueba.** Los 1,408,395 pares de `probe.txt` se sacan del entrenamiento y son el conjunto de prueba. Los
  hiperparámetros se eligen con una **validación tipo probe**: el último rating de cada usuario.
- **Submuestra S.** Las 5,000 películas más calificadas (93.5 % de los ratings) × 100,000 usuarios al azar: 19.3
  millones de ratings y 262,954 del probe. Los métodos que no escalan corren ahí, y todos los modelos se comparan sobre
  su probe. S excluye el 10.25 % del probe de esos usuarios (películas fuera del top 5,000).
- **Respaldo.** Si un modelo no puede predecir (usuario sin ratings en S, película sin vecinos), usa el baseline.
- **Top-N.** Se evalúa sobre 5,000 usuarios de S; son relevantes las películas del probe con rating ≥ 4 y se reportan
  P@10, R@10 y NDCG@10. Con ~2 relevantes por usuario, la P@10 máxima posible es 0.207.
- **Implementados por nosotros:** ALS de los sesgos, kNN, SGD, NMF multiplicativa, A-Priori, PCY, Multistage,
  Multihash, SON, Toivonen, PrefixSpan, PageRank y el paseo P³. Se usan bibliotecas solo para `svds`, PCA, t-SNE,
  TF-IDF y FP-Growth (`mlxtend`, como referencia).
- **Modelo adicional:** un ensamble ridge de los predictores (opcional), porque el curso no cubre la combinación de
  modelos.
- **Limitaciones:** no se modela la dinámica temporal (el salto del rating medio de 2004, Koren 2009); el catálogo solo
  trae título y año; las fechas son de calificación, no de visualización.

> Instituto de Investigaciones en Matemáticas Aplicadas y en Sistemas, Universidad Nacional Autónoma de México. Noviembre, 2026
