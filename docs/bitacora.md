# Bitácora del proyecto

Registro de lo que se ha hecho en cada paso, qué se ha descubierto y qué se ha decidido.

## Paso 1. Carga, uniones y variables prohibidas

**Qué se ha hecho.** Lectura de las 13 tablas con `\N` como valor ausente, filtrado de cualquier año posterior a 2024 (esta versión del dataset llega justo a 2024: 24 carreras, 479 resultados) y construcción de una tabla base con una fila por piloto y carrera a partir de `results.csv`, unida con `races`, `status`, `drivers` y `constructors`.

**Hallazgos**

- `raceId` no es cronológico: 2009 tiene los identificadores 1-17, 2008 los 18-35 y 1950 los 833-839. Todo cálculo de "carrera anterior" se hace con un índice cronológico propio (`race_idx`, ordenado por fecha).
- `results.csv` solo contiene Grandes Premios. Las sprints (desde 2021; 6 por temporada en 2023 y 2024) están en `sprint_results.csv`.
- Hay 91 duplicados piloto-carrera, todos hasta 1978 (coches compartidos). No afectan al periodo de trabajo.
- Desde 2014, de 4.626 resultados, 680 son abandonos (R), 28 retiradas antes de la salida (W) y 11 descalificaciones. `positionOrder` coloca a los no clasificados al final, por vueltas completadas. Solo un piloto no terminado acabó con `positionOrder ≤ 10` en ese periodo.
- `grid = 0` (salida desde el pit lane) aparece 79 veces desde 2014, con una posición final media de 14,8 frente a 10,6 del resto. Se recodificará como último de la parrilla + 1.
- La parrilla coincide con la posición de clasificación solo en el 68 % de los casos, por las penalizaciones. Ambas variables aportan información distinta.
- Tiempos de clasificación disponibles desde 1994. Se convierten a segundos y a distancia a la pole en %.
- Comprobado que `driver_standings` de una carrera es el mundial posterior: en Baréin 2023, los puntos del mundial coinciden exactamente con los puntos de esa carrera.

**Decisiones**

- Mundiales: se usa el de la carrera anterior de la misma temporada; en la ronda 1, cero.
- Lista de variables prohibidas fijada en el notebook (sección 3) y en la constante `PROHIBIDAS`, que se usará para comprobar automáticamente el conjunto de variables del modelo.
- Los abandonos se agrupan en terminado, fallo mecánico y accidente u otros, para construir variables de fiabilidad con carreras anteriores.

## Paso 2. EDA

**Qué se ha hecho.** Análisis exploratorio con datos hasta 2023 (sin mirar 2024, para no usar el test en ninguna decisión). Se responde a las cuatro preguntas del enunciado con un gráfico y una tabla cada una.

**Hallazgos**

- **La parrilla explica mucho.** Entre 2014 y 2023, el 81 % de los poleman terminan en el podio, y salir cuarto o quinto reduce esa probabilidad del 33 % al 15 %. El top 10 baja de forma más gradual: salir décimo da un 54 % de puntuar. Un modelo que predice "cada uno termina donde sale" logra un Spearman medio de 0,60 en 2014-2023 (0,598 en 2023). Es la cifra que hay que superar en la Parte 2.
- **Los abandonos han cambiado mucho.** Antes de los 2000 no terminaba la mitad de la parrilla; desde 2014, entre el 13 % y el 24 %. Las averías mecánicas bajaron del 14,5 % en 2014 (primer año de motores híbridos) a un 6-9 % en 2021-2023. Los accidentes se mantienen en torno al 7-8 %.
- **El circuito importa.** El Spearman parrilla-resultado va de alrededor de 0,4 (Hockenheim, Austin) a más de 0,7 (Shanghái, Paul Ricard, Abu Dabi). Contando solo a los que terminan, Mónaco es el circuito donde menos cambia el orden (0,86). En urbanos como Bakú o Singapur el desorden viene sobre todo de abandonos.
- **El coche domina.** Mercedes ganó entre el 52 % y el 90 % de las carreras de 2014 a 2020; Red Bull, el 95 % en 2023. El nivel de los equipos se mantiene de un año a otro (correlación de 0,83 a 0,93, salvo 0,72 en 2015).

**Decisiones**

- **Entrenar con las filas de 2014 a 2023** (4.147 resultados). Motivos: misma era de motores (híbridos), tasa de abandonos y Spearman de la parrilla estables, y sistema de puntos para los 10 primeros vigente desde 2010. Las variables históricas sí pueden usar carreras anteriores a 2014.
- **Validación:** entrenar con 2014-2022 y medir en 2023.
- **Variables que se crearán** a raíz del EDA: Spearman histórico del circuito, fiabilidad reciente del coche, forma reciente del equipo y del piloto, y mundial previo de constructores.

## Paso 3. Ingeniería de variables

**Qué se ha hecho.** 36 variables organizadas en siete bloques, cada uno en una función (`añadir_...`) para poder repetir el proceso entero y comprobar que no hay fugas. Las históricas se calculan sobre todo el histórico desde 1950, aunque el modelo solo aprenderá con las filas de 2014-2023.

| Bloque | Variables |
|---|---|
| Fin de semana | parrilla corregida, salida desde boxes, posición y distancia a la pole en clasificación, penalización (parrilla − clasificación), diferencia con el compañero en clasificación, posición en la sprint |
| Forma del piloto | posición media (5 y 10 carreras), puntos, % top 10, % podio, parrilla media, puestos ganados, % abandonos, experiencia |
| Equipo | posición media y puntos (5 carreras), % de averías (10 carreras) |
| Mundial previo | puntos, posición y victorias del piloto, % de los puntos del líder, puntos y posición del equipo |
| Compañero | % de duelos ganados en las últimas 10 carreras |
| Circuito | posición media del piloto en sus últimas visitas, número de visitas, Spearman histórico parrilla-resultado del circuito |
| Relativas | posición de cada piloto dentro de la carrera en forma del equipo, forma del piloto, mundial y distancia a la pole |

**Decisiones**

- **Linajes de equipo.** Los equipos que cambian de nombre se agrupan (Force India → Racing Point → Aston Martin; Toro Rosso → AlphaTauri → RB; Lotus F1 → Renault → Alpine; Sauber ↔ Alfa Romeo; Marussia → Manor), para que su forma no se reinicie con el cambio de nombre. Para el mundial de constructores se mantiene el `constructorId` original, que es como lo publica la tabla.
- **Distancia a la pole limitada al 10 %**, para que un piloto que apenas rueda en clasificación no distorsione el modelo.
- **Mundial**: se toma de la carrera anterior de la misma temporada; en la ronda 1, puntos a cero y posición vacía.
- **Circuitos nuevos** sin historial: `circuito_spearman` toma la media de todas las carreras anteriores.

**Comprobaciones de fuga**

1. Ninguna variable está en la lista de prohibidas (`assert`).
2. Comprobación a mano: la media de las 5 carreras anteriores de Alonso antes de Mónaco 2023 (3,2) coincide con `piloto_pos_5`.
3. La suma de puntos de mundial en todas las rondas 1 es 0.
4. **Prueba de alteración:** se desordenan al azar los resultados del GP de Italia 2023 y se reconstruyen todas las variables. Las de esa carrera quedan idénticas y las de carreras posteriores cambian, así que ninguna variable usa el resultado de la carrera que se predice.

**Primeros indicios** (Spearman de cada variable con la posición final, 2014-2023): las más relacionadas son la clasificación (0,61-0,63), la forma del equipo (0,59-0,60), el mundial (0,59-0,60) y la forma del piloto (0,56-0,59). Las del circuito o el compañero tienen poca relación por sí solas; el modelo dirá si aportan combinadas con las demás.

**Datos para modelar:** 4.626 filas, 4.147 de entrenamiento (2014-2023) y 479 de test (2024).

## Paso 4. Validación temporal, métricas y referencia de parrilla

**Qué se ha hecho.**

- **Validación temporal**: se entrena con 2014-2022 (3.707 filas, 182 carreras) y se mide con 2023 (440 filas, 22 carreras). No se divide al azar, para que el modelo nunca vea carreras posteriores a las que se le piden.
- **Función `evaluar`**: calcula las métricas del reto (log loss de podio y de top 10 y su media, Spearman medio por carrera y MAE) y comprueba que `p_podio ≤ p_top10`.
- **Dos referencias**, medidas en 2023:

| Modelo | Log loss podio | Log loss top 10 | Parte 1 (media) | Parte 2 (Spearman) | MAE |
|---|---|---|---|---|---|
| Sin información | 0,4227 | 0,6933 | 0,5580 | −0,03 | 6,67 |
| Solo parrilla | 0,2514 | 0,5302 | **0,3908** | **0,5973** | 3,71 |

La referencia de parrilla usa `posicion_pred = grid_adj` y una regresión logística con la parrilla y su logaritmo para las probabilidades.

**Hallazgos**

- **Empates en la Parte 2.** Dando la misma `posicion_pred` a todos, el Spearman salía 0,25 en lugar de 0, porque los empates se resuelven por el orden de las filas. La entrega no debe tener empates en `posicion_pred`.
- **Sumas por carrera.** Con la referencia, las `p_podio` de cada carrera de 2023 suman de media 2,98 (entre 2,35 y 3,01) y las `p_top10`, 9,98 (entre 9,19 y 10,17). Se valorará reescalarlas a 3 y 10 con los modelos completos.

**Notas a batir en validación**: log loss media por debajo de 0,3908 y Spearman por encima de 0,5973.

## Paso 5. Parte 1: probabilidades

**Qué se ha hecho.** Se han comparado, con validación en 2023, una regresión logística y un LightGBM con las 36 variables, su media y un ajuste para que las probabilidades de cada carrera sumen 3 (podio) y 10 (top 10).

| Modelo (validación 2023) | Log loss podio | Log loss top 10 | Media |
|---|---|---|---|
| Solo parrilla | 0,2514 | 0,5302 | 0,3908 |
| Logística (C = 0,03) | 0,2225 | 0,4631 | 0,3428 |
| LightGBM (árboles de 4 hojas, 600 árboles) | 0,2263 | 0,4634 | 0,3448 |
| Media de los dos | 0,2206 | 0,4612 | 0,3409 |
| **Media de los dos + ajuste 3/10** | **0,2164** | **0,4610** | **0,3387** |

**Decisiones**

- **Hiperparámetros.** Logística con `C = 0,03` (la mejor de cinco probadas). LightGBM con árboles pequeños (4 hojas, 600 árboles, `learning_rate` 0,01): los árboles grandes sobreajustaban (0,3578 con 15 hojas).
- **Modelo elegido:** media de las probabilidades de la logística y de LightGBM, reescaladas por carrera a 3 y 10 (limitadas entre 0,0001 y 0,995), y `p_top10 = max(p_top10, p_podio)`.
- **Robustez.** Repetida la comparación validando en 2021 y 2022: el modelo elegido mejora a la parrilla en 0,04-0,05 todos los años y es el mejor o casi (en 2022 la logística ajustada gana por 0,002).

**Hallazgos**

- Las variables con más peso son las de clasificación (parrilla, posición y distancia a la pole), seguidas de la forma del equipo y del piloto. La posición en la sprint también aparece entre las diez primeras.
- Las probabilidades están razonablemente calibradas: los pilotos a los que se dio entre un 50 y un 70 % de podio lo lograron en un 57,7 % de los casos (57,8 % predicho).

## Paso 6. Parte 2: posición

**Qué se ha hecho.** Tres formas de predecir el orden, más dos combinaciones, con validación en 2023 y comprobación en 2021 y 2022. Los empates se deshacen con la parrilla (se suma una millonésima por puesto de salida).

| Modelo | Spearman 2023 | MAE 2023 | Spearman medio 2021-2023 |
|---|---|---|---|
| Solo parrilla | 0,5973 | 3,71 | 0,5907 |
| Regresión (Ridge, `alpha` = 10) | 0,6656 | 3,25 | 0,6540 |
| Ranking (LGBMRanker, lambdarank) | 0,6651 | 3,29 | 0,6557 |
| Desde probabilidades (`p_podio + p_top10`) | 0,6620 | 3,32 | 0,6533 |
| **Regresión + ranking (suma de órdenes)** | **0,6683** | **3,25** | **0,6563** |
| Regresión + ranking + probabilidades | 0,6637 | 3,29 | 0,6532 |

**Decisiones**

- **Modelo elegido:** suma del orden dentro de la carrera que da la regresión Ridge y del que da el LGBMRanker. Es la mejor media de los tres años y la mejor en 2021 y 2023.
- **Ranker:** relevancia = número de coches − posición final; árboles de 4 hojas, 600 árboles, `learning_rate` 0,02.
- No se añade la opción desde probabilidades a la combinación: usa la misma información y no mejora.

**Hallazgos**

- Todas las opciones superan a la parrilla entre 0,05 y 0,09 puntos cada año. Las diferencias entre ellas son de milésimas y cambian según el año.
- En Singapur 2023 el modelo coloca 2.º a Verstappen (salía 11.º, terminó 5.º) y 8.º a Pérez (salía 13.º, terminó 8.º).

## Paso 7. Análisis de errores

**Qué se ha hecho.** Revisión de los errores del modelo elegido de la Parte 2 en la validación de 2023, con los tres focos que propone el reto. Los datos de la propia carrera (abandonos, tiempos por vuelta) se usan solo para examinar los errores, nunca como variables.

**Hallazgos**

- **Abandonos: la mayor fuente de error.** El error medio es de 2,65 puestos con los pilotos que terminan y de 6,67 con los que no. Sin los que abandonan, el Spearman medio pasaría de 0,668 a 0,798. Las carreras con más abandonos son las peor predichas (correlación de −0,69): Estados Unidos (0,17, con Hamilton y Leclerc descalificados) y Australia (0,26, con 8 pilotos sin terminar).
- **Lluvia: no se puede detectar con estos datos.** Se probó como indicador el cociente entre la vuelta mediana de carrera y la pole. Va de 1,06 a 1,12 y no tiene relación con el error (correlación de 0,03), porque mezcla gasolina, neumáticos, coches de seguridad y lluvia del sábado. Canadá sale por debajo de 1, lo que apunta a una clasificación en mojado.
- **Cambio de nivel entre temporadas: efecto pequeño.** Aston Martin mejoró casi 3 puestos de media de 2022 a 2023 y el modelo solo se equivoca 0,6 puestos con él en las seis primeras carreras: la clasificación del fin de semana recoge el cambio enseguida.
- **Ritmo de clasificación frente a ritmo de carrera.** El modelo sobrestima a Ferrari (−4,3 puestos en las rondas 1-6, −2,9 después) y a Mercedes (−2,3 desde la ronda 7). Clasificaban mejor de lo que rendían en carrera.
- **A lo largo de la temporada** el modelo supera a la parrilla en todos los tramos. El más difícil es el de las rondas 13-18.

**Mejora posible (no implementada):** variables de ritmo de carrera a partir de `lap_times` de carreras anteriores.

## Paso 8. Entrega

**Qué se ha hecho.** Los modelos elegidos en validación se reentrenan con 2014-2023 (4.147 filas) y se aplican a las 24 carreras de 2024 (479 filas). Se exporta `predicciones_2024.csv` con `resultId`, `raceId`, `driverId`, `p_podio`, `p_top10` y `posicion_pred`.

**Comprobaciones automáticas** (`assert`): una fila por cada resultado de 2024 sin repetidos, sin vacíos, probabilidades entre 0 y 1, `p_podio ≤ p_top10` en todas las filas, y `posicion_pred` como orden 1…n sin empates en cada carrera. Las `p_podio` suman 3 en cada carrera; las `p_top10`, entre 10,000 y 10,094.

**Nota en 2024** (calculada después de exportar, solo informativa; no se ha usado para cambiar nada):

| Modelo | Log loss podio | Log loss top 10 | Parte 1 | Parte 2 (Spearman) | MAE |
|---|---|---|---|---|---|
| Solo parrilla | 0,2366 | 0,4379 | 0,3373 | 0,7286 | 2,90 |
| **Entrega** | **0,2284** | **0,3919** | **0,3101** | **0,7554** | **2,87** |

La entrega supera a la referencia en las dos partes. En 2024 la parrilla explicó más que en 2023 (0,73 frente a 0,60), por eso las cifras de ambos son mejores que en validación.

**Reproducibilidad.** Ejecutado de principio a fin dos veces con `SEED = 2502`: el CSV sale idéntico.
