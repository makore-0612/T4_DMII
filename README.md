<img src="https://mmss.iimas.unam.mx/dmmss/wp-content/uploads/2026/02/cropped-LogoUNAM_IIMAS_Negro50-scaled-2.png" width="200"/>

### Datos Masivos II
# Práctica 4: Sistemas de recomendación
> Equipo Tres: Castrillo Cruz Karen Arlet, Ramos González Nadia, Zamora Antiga Ángel Javier.

---

Este repositorio construye y evalúa sistemas de recomendación sobre dos conjuntos de datos de escala y naturaleza muy
distintas: uno de **calificaciones densas** de usuarios a películas y otro de **mediciones dispersas** de afinidad
entre proteínas y moléculas. En los dos, la matriz usuario × ítem es la misma idea; lo que cambia es qué tan vacía
está y qué se puede hacer cuando el ítem es nuevo.

| Carpeta | Dataset | Matriz | Usuario × Ítem | Calificación |
|---|---|---|---|---|
| `NetflixPrize/` | [Netflix Prize (Kaggle)](https://www.kaggle.com/datasets/netflix-inc/netflix-prize-data): ~100 millones de ratings, 1999–2005 | 480,189 × 17,770, densidad 1.18 % | usuario × película | entero de 1 a 5 |
| `BindingDB/` | [BindingDB](https://www.bindingdb.org), volcado `BindingDB_All` release 202611: 3,243,660 mediciones de afinidad | 3,444 × 832,751, densidad 0.044 % | proteína × ligando | `pIC50` continuo, y actividad binaria |

```text
T4_DMII/
├── NetflixPrize/
│   ├── notebooks/
│   │   ├── 01_carga_eda.ipynb              # Parquet por bloques, CSR, EDA, probe, submuestra S
│   │   ├── 02_baseline_vecindarios.ipynb   # validación tipo probe, baseline, kNN ítem-ítem (RMSE y top-N)
│   │   ├── 03_factorizacion.ipynb          # SVD truncada, PureSVD, SVD-SGD, NMF, PCA / t-SNE
│   │   ├── 04_reglas_secuencias.ipynb      # itemsets frecuentes (7 algoritmos), reglas y PrefixSpan
│   │   ├── 05_grafos_contenido.ipynb       # grafo de películas, PageRank, PPR, P³ bipartito, contenido
│   │   └── 06_comparacion.ipynb            # tabla comparativa, ensamble, ejemplos, conclusiones
│   ├── requirements.txt
│   ├── README.md                           # descarga de los datos, resultados y orden de ejecución
│   └── data/                               # no incluida: ver NetflixPrize/README.md
└── BindingDB/
    ├── notebooks/
    │   ├── 01_eda.ipynb                    # lectura del TSV de 9 GB, calidad por columna y sesgos de fuente
    │   ├── 02_limpieza.ipynb               # matriz de interacciones, pIC50, actividad binaria y cotas
    │   ├── 03_recomendador.ipynb           # huellas y k-meros, partición temporal, 5 modelos y comparación
    │   └── 04_pares_nuevos.ipynb           # pares sin medir, validación por dilución, catálogo y filtro
    ├── requirements.txt
    ├── README.md                           # descarga de los datos, resultados y orden de ejecución
    ├── data/                               # no incluida: ver BindingDB/README.md
    └── figs/                               # no incluida: la generan los notebooks
```

## Qué hay en cada carpeta

**`NetflixPrize/`** recorre los métodos del curso sobre los 100 millones de ratings y los compara todos contra el
conjunto oficial `probe.txt`: filtrado colaborativo, SVD, NMF, PCA, t-SNE, A-Priori, PCY, Multistage, Multihash, SON,
Toivonen, FP-Growth, PrefixSpan, PageRank por el método de potencias, grafos y recomendación basada en contenido. El
**SVD con sesgos por SGD**, implementado por el equipo, llega a un RMSE de **0.9130** en el probe completo: **3.6 %
mejor que Cinematch**. En top-N ganan los modelos que acumulan evidencia y corrigen la popularidad, no los de menor
RMSE.

**`BindingDB/`** construye un recomendador **híbrido** donde la proteína es el usuario y la molécula el ítem. El caso
difícil no es el borde: el 72.6 % de los ligandos se midió contra una sola proteína, así que el colaborativo clásico
cubre apenas el 1.5 % de los pares y **el 0 % de las moléculas nuevas**. El híbrido —contenido químico, memoria
colaborativa y proteínas homólogas— cubre el 99.8 % y recupera el 16.3 % del margen disponible sobre el azar. El
notebook 04 lleva el modelo a los pares que nadie ha medido, el 99.96 % de la matriz.

Los dos README explican por qué la **partición importa más que el modelo**: al azar, los análogos casi idénticos de un
mismo artículo (BindingDB) y los ratings posteriores del mismo usuario (Netflix) filtran la prueba al entrenamiento.
BindingDB lo cuantifica: partir al azar infla la ventaja sobre el azar en **+316 %**.

## Cómo replicarlo

Los datos no se incluyen en el repositorio: `data/` y `figs/` están en el `.gitignore`. Cada carpeta explica cómo
obtenerlos y tiene su propio `requirements.txt` con las versiones con las que se ejecutaron sus notebooks. Ninguno de
los notebooks de BindingDB tiene salidas guardadas; hay que ejecutarlos en orden.

| Carpeta | Datos | Peso | Cómo se consiguen |
|---|---|---|---|
| `NetflixPrize/` | cuatro `combined_data_*.txt`, `probe.txt`, `qualifying.txt`, `movie_titles.csv` | ~2.1 GB | `kaggle datasets download -d netflix-inc/netflix-prize-data -p data/ --unzip` |
| `BindingDB/` | `BindingDB_All_202611.tsv.zip` | 9 GB sin comprimir | [página de descarga de BindingDB](https://www.bindingdb.org/rwd/bind/chemsearch/marvin/Download.jsp), o pedir los Parquet al equipo |

> Instituto de Investigaciones en Matemáticas Aplicadas y en Sistemas, Universidad Nacional Autónoma de México.
> Noviembre, 2026
