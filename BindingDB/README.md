# T4_DMII

Datos Masivos II (IIMAS–UNAM): Sistemas de recomendación híbridos.

**Equipo Tres:** Castrillo Cruz Karen Arlet, Ramos González Nadia, Zamora Antiga Ángel Javier.

Dataset: [BindingDB](https://www.bindingdb.org), volcado completo `BindingDB_All` (release 202611), 3,243,660 mediciones de afinidad entre proteínas y moléculas pequeñas.

**La tarea.** Recomendar, para una proteína humana dada, qué moléculas pequeñas (*ligandos*) tienen más probabilidad de unirse a ella con fuerza. La proteína hace de **usuario**, el ligando de **ítem** y la potencia medida (`pIC50`) de **calificación**.

## Contenido

Se corren en este orden; cada uno lee lo que escribió el anterior.

| Notebook | Qué hace | Tarda |
|---|---|---|
| `BindingDB_EDA.ipynb` | Análisis exploratorio. Lee el TSV de 9 GB por bloques y lo deja tipificado en parquet. Solo descriptivo. | ~5 min la primera vez, luego cacheado |
| `BindingDB_Limpieza.ipynb` | Construye la matriz de interacciones lista para el sistema de recomendación. | <2 min |
| `BindingDB_Recomendador.ipynb` | **El recomendador.** Construye las representaciones, parte los datos, entrena los cinco modelos y los compara contra el azar. | ~7 min (~13 la primera vez) |
| `BindingDB_Recomendador_ParesNuevos.ipynb` | El mismo modelo usado para lo que **nadie ha medido**: valida que el ranking sirva fuera de los pares con verdad conocida y recomienda sobre un catálogo. | ~9 min (~13 la primera vez) |

Las figuras quedan en `figs/eda/`, `figs/limpieza/`, `figs/recomendador/` y `figs/pares_nuevos/`. La diferencia entre el tiempo de la primera corrida y el de las siguientes es la caché de huellas moleculares (`fp_morgan_r2_2048.npz`, 66 MB), que tarda ~6 min en calcularse y la escribe el primero de los dos notebooks que se ejecute.

**Ninguno de los notebooks tiene salidas guardadas.** Hay que ejecutarlos.

## Los datos no están en el repositorio

`data/` está en `.gitignore`: el TSV original pesa 9 GB y los parquet intermedios, cientos de MB. **Clonar el repo no trae los datos.** Hay dos maneras de conseguirlos:

1. Pedirlos al equipo (se comparten por separado).
2. Regenerarlos: descargar `BindingDB_All_202611.tsv.zip` de la [página de descarga](https://www.bindingdb.org/rwd/bind/chemsearch/marvin/Download.jsp), descomprimirlo en `data/BindingDB_All_202611_tsv/` y correr los notebooks en orden.

Entorno: `pandas`, `numpy`, `pyarrow`, `matplotlib`, `scipy`, `scikit-learn` y `rdkit`. Dos advertencias de instalación:

- `scikit-learn` tiene que ser **≥ 1.6** si `numpy` es 2.x. Con una versión anterior el `import` falla con `ValueError: numpy.dtype size changed, may indicate binary incompatibility` — está compilada contra numpy 1.x.
- `rdkit` solo lo necesitan los dos notebooks del recomendador (huellas moleculares y dibujo de estructuras). El EDA y la limpieza corren sin él.

## Entrega de la limpieza

`BindingDB_Limpieza.ipynb` escribe cuatro archivos en `data/processed/`:

| Archivo | Filas | Una fila por |
|---|---|---|
| `rec_interacciones.parquet` | 1,250,494 | par (proteína, ligando) — **la matriz** |
| `rec_ligandos.parquet` | 832,751 | ligando — atributos de ítem |
| `rec_targets.parquet` | 3,444 | proteína — atributos de usuario |
| `rec_meta.json` | — | parámetros y embudo de la corrida |

La matriz tiene una densidad de 0.044% y un 61.9% de pares activos.

### `rec_interacciones.parquet`

| Columna | Significado |
|---|---|
| `target_id` | **Usuario.** Accession de UniProt. Une con `rec_targets.parquet`. |
| `lig_id` | **Ítem.** InChIKey recortado a 25 caracteres. Une con `rec_ligandos.parquet`. |
| `pIC50` | **Rating continuo.** `9 − log₁₀(IC50 nM)`; más grande = más potente. Mediana de las mediciones exactas. **Nulo** en el 20.7% de los pares, los que solo tienen cotas. |
| `activo` | **Rating binario.** 1 si `pIC50 ≥ 6` (1 µM), o si las cotas lo concluyen. Nunca es nulo. |
| `n_mediciones`, `n_exactas`, `n_censuradas` | Cuántas veces se midió el par y de qué tipo. Las dos últimas suman la primera. |
| `pIC50_min`, `pIC50_max`, `rango_log` | Dispersión de las mediciones exactas del par. |
| `discordante` | `rango_log > 1`: las mediciones del par difieren en más de un orden de magnitud. |
| `solo_censurado` | El par no tiene valor puntual; usar `activo`, no `pIC50`. |
| `conflicto_cotas` | Cotas que se contradicen. Son 1,148; conviene excluirlos. |
| `anio_min`, `anio_max` | Año de publicación. **Para la partición temporal.** Nulo en ~3% de los pares. |
| `alguna_patente` | Al menos una medición viene de una patente. |

`rec_ligandos.parquet`: `lig_id`, `smiles`, `inchikey`, `nombre`, `id_cid`, `id_chembl`, `n_targets`, `n_mediciones`.

`rec_targets.parquet`: `target_id`, `entry_name`, `nombre`, `secuencia`, `longitud`, `n_nombres_bdb`, `n_ligandos`, `n_mediciones`.

### Cuatro cosas que conviene saber antes de modelar

1. **El arranque en frío es el caso típico, no el borde.** El 72.6% de los ligandos se midió contra un solo target, así que no tienen con quién compararse por co-ocurrencia. Es la razón por la que el modelo tiene que ser híbrido: la parte de contenido (`smiles`, `secuencia`) es lo único que puede decir algo sobre ellos.
2. **Una partición aleatoria infla los resultados.** Un mismo artículo publica decenas de análogos casi idénticos; repartirlos al azar deja copias del test dentro del train. Conviene partir por tiempo (`anio_min`) o por ligando nuevo (`lig_id`), y reportar los pares de arranque en frío aparte del promedio global.
3. **Hay un techo para el RMSE.** El 19.8% de los pares medidos más de una vez tiene un rango mayor a 1 unidad de pIC50. Ese es ruido experimental, no error del modelo: no tiene sentido perseguir un RMSE por debajo de ~0.5.
4. **El tipo de medición está confundido con la fuente.** IC50 es el formato de las patentes (EDA §6.1). Quedarse solo con IC50 hereda ese sesgo y la limpieza no puede quitarlo; por eso cada par lleva `alguna_patente` y el año.

Los detalles y la justificación de cada decisión están en `BindingDB_Limpieza.ipynb`, que cita la sección del EDA donde se diagnosticó cada cosa. Los parámetros de limpieza (tipo de medición, especie, umbral de actividad) están juntos en una sola celda al inicio, por si hay que rehacer la entrega con otro criterio.

## El recomendador

`BindingDB_Recomendador.ipynb` trabaja sobre 1,248,252 pares, 3,444 proteínas y 830,999 ligandos — densidad **0.04362%**.

### Cómo se vuelven números

- **Moléculas:** ECFP4 (huella de Morgan, radio 2, 2048 bits). Dos moléculas se comparan con **Tanimoto** sobre sus bits.
- **Proteínas:** conteos de **3-meros** de la secuencia de aminoácidos (8,000 dimensiones), coseno entre ellos. Es bolsa de palabras: aproxima la identidad de secuencia sin alinear. La vecina más parecida de una proteína pertenece a su misma familia el **52.2%** de las veces.

### Los cinco modelos y las dos líneas base

| | Qué usa |
|---|---|
| `azar` | nada — permutación aleatoria |
| `media_proteina` | solo el promedio de la proteína (ordena igual todas sus moléculas) |
| `popularidad` | cuántas veces se midió cada molécula y con qué resultado |
| `colaborativo` | co-ocurrencia: moléculas medidas contra las mismas proteínas |
| `contenido` | similitud química (Tanimoto) con lo que esa proteína ya une |
| **`híbrido`** | contenido + colaborativo + proteínas homólogas, con dos pesos ajustados en validación |

El híbrido combina tres fuentes con `λ` (cuánto pesa la memoria colaborativa frente al contenido) y `γ` (cuánto pesa lo que unen las proteínas parecidas). Se barrieron `λ ∈ {0, .5, 1, 2, 3, 5}` y `γ ∈ {0, 1, 3, 10}`; ganan **λ = 2, γ = 3**, con +2.74% de NDCG sobre contenido puro.

### Resultados

Partición **temporal**: entrena con lo publicado hasta 2022 (1,040,255 pares), evalúa con lo publicado después (107,399 pares). El **97.3%** de los pares de test tiene una molécula que el modelo nunca vio. 668 proteínas evaluadas.

El azar acierta 0.632 porque **el 63% de los pares medidos son activos**: los químicos prueban lo que esperan que funcione. Por eso el enriquecimiento tiene un techo de **2.320×**, no infinito, y la columna que importa es cuánto del margen disponible se recupera.

| modelo | precision@10 | NDCG@10 | enriquecimiento | % del margen | AUC | RMSE |
|---|---|---|---|---|---|---|
| azar | 0.636 | 0.657 | 1.001× | 0.1% | 0.504 | — |
| media_proteina | 0.631 | 0.652 | 0.984× | −1.2% | 0.500 | 1.190 |
| popularidad | 0.638 | 0.660 | 1.014× | 1.0% | 0.508 | 1.228 |
| colaborativo | 0.631 | 0.655 | 1.029× | 2.2% | 0.501 | 1.339 |
| contenido | 0.690 | 0.719 | 1.158× | 12.0% | 0.598 | 1.139 |
| **híbrido** | **0.700** | **0.737** | **1.216×** | **16.3%** | **0.617** | **1.112** |

La cobertura explica por qué el híbrido existe:

| modelo | global | molécula ya vista | molécula nueva |
|---|---|---|---|
| colaborativo | 1.5% | 53.6% | **0.0%** |
| contenido | 91.0% | 87.5% | 91.1% |
| **híbrido** | **99.8%** | 99.7% | **99.8%** |

El colaborativo clásico no es malo: **no opina**. En el 98.5% de los casos no tiene con quién comparar, y en una molécula nueva nunca puede.

Tres cosas más que el notebook mide y vale la pena no olvidar:

- **La partición aleatoria infla la ventaja sobre el azar en +316%** (1.758× contra 1.216×). Es el número 2 de la lista de arriba, cuantificado.
- **`media_proteina` consigue una correlación agrupada de r = 0.300 con p ≈ 0 y, al mismo tiempo, un AUC de exactamente 0.500.** Ordena igual todas las moléculas de cada proteína: no distingue nada. La correlación agrupada mide que el modelo sabe *qué proteínas son fáciles*, no que sepa *qué molécula recomendar*. Hay que mirar la correlación dentro de cada proteína (híbrido: 0.233, positiva en el 72% de 611 proteínas).
- **El RMSE de 1.112 está lejos del piso de ~0.5**, pero el piso es ruido de laboratorio, no una meta alcanzable.

Escribe `data/processed/rec_resultados.json` (todos los números de arriba) y siete figuras en `figs/recomendador/`.

## Recomendar pares que nadie ha medido

`BindingDB_Recomendador_ParesNuevos.ipynb` responde la objeción obvia: la evaluación de arriba solo usa pares medidos, porque son los únicos con verdad conocida. Pero el objetivo es recomendar lo que **no** se ha medido, y eso es el **99.96%** de la matriz: 2,861,960,556 celdas, de las cuales 1,248,252 tienen medición y **2,860,712,304** no.

El notebook reorganiza el modelo como objeto que se ajusta una vez (`ajustar`) y luego puntúa cualquier par, medido o no (`puntuar`), y lo valida de tres formas:

1. **Retrospectiva** — entrena hasta 2022 y puntúa lo que se midió después: `precision@10` 0.705, enriquecimiento **1.214×** de un techo de 2.320×, AUC 0.618. Consistente con el notebook principal.
2. **Dilución** — por cada candidato con verdad conocida se meten 20 moléculas sin medir (300 + 6,000 por proteína, 8 proteínas). Por azar se esperarían 0.48 candidatos con verdad en el top-10; se observan 4.62, o **9.7× de concentración**. El resultado es bimodal: en 4 proteínas los candidatos con verdad copan el top-10 (8–10 de 10), en las otras 4 no entra ninguno. No es inestabilidad: depende de si los candidatos con verdad de esa proteína tienen análogos en el entrenamiento. Cuando no los tienen, el modelo prefiere con razón moléculas del catálogo que sí se parecen a lo que esa proteína une.
3. **Catálogo nunca medido** — 29,898 moléculas candidatas × 8 proteínas, 35 s de cómputo, 80 recomendaciones.

Y deja clara la trampa de leer esos números: **solo el 22% de las recomendaciones del catálogo cae dentro del dominio de aplicabilidad** (Tanimoto máximo ≥ 0.30 con algo que esa proteína ya une). Tomar moléculas al azar de entre 831 mil garantiza que casi ninguna se parezca a lo que une una proteína concreta. Para usar esto de verdad, el catálogo debería construirse por **búsqueda por similitud** desde los activos conocidos, no muestrearse al azar; el muestreo aleatorio sirve aquí para exhibir el mecanismo y el filtro funcionando.

Lo mismo vale para el `precision@10`: no se transfiere. Entre los pares *medidos* el 63% es activo; entre los no medidos la tasa real de aciertos es órdenes de magnitud menor. Lo único que se traslada es el **enriquecimiento relativo**: el modelo ordena mejor que el azar, no promete que 7 de cada 10 recomendaciones funcionen.

Escribe `data/processed/rec_pares_nuevos.json`, `data/processed/recomendaciones_pares_nuevos.csv` (una fila por recomendación, con `pIC50_pred`, `tanimoto_max`, `n_fuentes` y `dentro_del_dominio`) y dos figuras en `figs/pares_nuevos/`.

## Glosario mínimo

| Término | Qué es |
|---|---|
| **ligando** | la molécula pequeña; el **ítem** del sistema de recomendación |
| **proteína** / *target* | la proteína humana a la que se busca algo que se le una; el **usuario** |
| **IC50** | concentración de molécula que inhibe la mitad de la actividad de la proteína. Más chica = más potente |
| **pIC50** | `9 − log₁₀(IC50 nM)`. Más grande = más potente. 6 = 1 µM, 9 = 1 nM. Un cambio de 10× en concentración es siempre exactamente 1 unidad, y 1 unidad ≈ 1.36 kcal/mol de energía libre de unión |
| **activo** | `pIC50 ≥ 6`: se une por debajo de 1 µM. **No** significa que active la proteína — IC50 mide adherencia, no función |
| **SMILES** | cadena de texto que codifica la topología 2D de una molécula. No es única, por eso la limpieza elige la más frecuente por `lig_id` |
| **ECFP4** | huella molecular: vector binario donde cada bit marca la presencia de un subfragmento de la molécula |
| **Tanimoto** | parecido entre dos huellas: bits en común / bits en al menos una. Entre 0 y 1 |
| ***k*-mero** | subcadena de longitud *k* de una secuencia de aminoácidos. Las cuentas de 3-meros son la bolsa de palabras de una proteína |
| **dominio de aplicabilidad** | una predicción solo es creíble si la molécula consultada se parece a algo que el modelo ya vio |
