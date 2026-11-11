# T4_DMII

Datos Masivos II (IIMAS–UNAM): Sistemas de recomendación híbridos.

**Equipo Tres:** Castrillo Cruz Karen Arlet, Ramos González Nadia, Zamora Antiga Ángel Javier.

Dataset: [BindingDB](https://www.bindingdb.org), volcado completo `BindingDB_All` (release 202611), 3,243,660 mediciones de afinidad entre proteínas y moléculas pequeñas.

## Contenido

| Notebook | Qué hace |
|---|---|
| `BindingDB_EDA.ipynb` | Análisis exploratorio. Lee el TSV de 9 GB por bloques y lo deja tipificado en parquet. Solo descriptivo. |
| `BindingDB_Limpieza.ipynb` | Construye la matriz de interacciones lista para el sistema de recomendación. |

Las figuras de cada uno quedan en `figs/eda/` y `figs/limpieza/`.

## Los datos no están en el repositorio

`data/` está en `.gitignore`: el TSV original pesa 9 GB y los parquet intermedios, cientos de MB. **Clonar el repo no trae los datos.** Hay dos maneras de conseguirlos:

1. Pedirlos al equipo (se comparten por separado).
2. Regenerarlos: descargar `BindingDB_All_202611.tsv.zip` de la [página de descarga](https://www.bindingdb.org/rwd/bind/chemsearch/marvin/Download.jsp), descomprimirlo en `data/BindingDB_All_202611_tsv/` y correr los dos notebooks en orden. El EDA tarda unos 5 minutos en leer el TSV (se cachea, así que solo la primera vez) y la limpieza, menos de 2.

Entorno: `pandas`, `numpy`, `pyarrow`, `matplotlib`. Para los *fingerprints* moleculares hace falta además `rdkit`, que **no** está instalado.

## Entrega para el sistema de recomendación

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
