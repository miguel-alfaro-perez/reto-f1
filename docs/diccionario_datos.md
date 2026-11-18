# Diccionario de datos

Qué significa cada columna que usamos y si se puede usar para predecir la carrera.

## La pregunta que decide si una columna se puede usar

Predecimos el **sábado por la noche**, después de la clasificación y antes de la carrera. Para cada dato hay que preguntarse:

> ¿Se conoce este dato el sábado por la noche?

- **Sí**: se puede usar.
- **No, porque se produce durante o después de la carrera**: está prohibido para esa carrera. Sí se puede usar el mismo dato de carreras **anteriores**, porque esas ya se han corrido.

## Ejemplo real: GP de Abu Dabi 2024

Dos filas de `results.csv`:

| Columna | Lando Norris | Sergio Pérez | ¿Se sabe el sábado? |
|---|---|---|---|
| `grid` | 1 | 10 | Sí, sale de la clasificación |
| `position` | 1 | (vacío) | No |
| `positionText` | 1 | R | No |
| `positionOrder` | 1 | 20 | No: es lo que queremos predecir |
| `points` | 25 | 0 | No |
| `laps` | 58 | 0 | No |
| `time` | 1:26:33.291 | (vacío) | No |
| `fastestLap` | 52 | (vacío) | No |
| `status` | Finished | Collision | No |

Con el sábado solo sabemos que Norris sale primero y Pérez décimo. Todo lo demás pasó el domingo. Si el modelo recibiera `laps = 0`, sabría que Pérez abandonó en la primera vuelta y acertaría su posición sin ningún mérito. Por eso esas columnas están prohibidas: **contienen la respuesta**.

## `results.csv`: una fila por piloto y Gran Premio

| Columna | Significado | ¿Se puede usar para esa carrera? |
|---|---|---|
| `resultId` | identificador de la fila | Solo como identificador, para el CSV de entrega |
| `raceId` | identificador de la carrera | Solo para unir tablas |
| `driverId` | identificador del piloto | Sí (quién es el piloto) |
| `constructorId` | identificador del equipo | Sí (en qué equipo corre) |
| `number` | dorsal del coche | Sí, pero no aporta nada |
| `grid` | posición de salida (0 = sale desde boxes) | **Sí**, es la variable más importante |
| `position` | posición oficial; vacía si no se clasifica | No: resultado de la carrera |
| `positionText` | posición o código de abandono (R, D, W…) | No: resultado de la carrera |
| `positionOrder` | posición final, siempre con valor | No: es el **objetivo** |
| `points` | puntos ganados en esa carrera | No: depende de la posición final |
| `laps` | vueltas completadas | No: revela si abandonó y cuándo |
| `time`, `milliseconds` | tiempo total o diferencia con el ganador | No: resultado de la carrera |
| `fastestLap`, `rank`, `fastestLapTime`, `fastestLapSpeed` | datos de su vuelta más rápida | No: se miden durante la carrera |
| `statusId` | código del estado final (terminó, motor, accidente…) | No: revela si abandonó |

## `qualifying.csv`: una fila por piloto y clasificación

| Columna | Significado | ¿Se puede usar? |
|---|---|---|
| `position` | puesto en la clasificación | Sí, se disputa el sábado |
| `q1`, `q2`, `q3` | mejor vuelta en cada ronda (Q2 solo los 15 mejores, Q3 los 10 mejores) | Sí, se disputa el sábado |

A partir de estas creamos `gap_pole_pct`: cuánto más lento fue el piloto que la pole, en porcentaje.

## `sprint_results.csv`: una fila por piloto y sprint

Mismas columnas que `results.csv`, pero de la sprint. Desde 2021, en algunos fines de semana. La sprint se corre **antes** del Gran Premio, así que su resultado **sí** se puede usar para predecir la carrera del domingo de ese fin de semana.

## `driver_standings.csv` y `constructor_standings.csv`: el mundial

| Columna | Significado |
|---|---|
| `points` | puntos acumulados en el mundial |
| `position` | posición en el mundial |
| `wins` | victorias acumuladas |

**La trampa.** La fila de una carrera es el mundial **después** de esa carrera: ya incluye sus puntos. En Baréin 2023, primera carrera del año, Verstappen aparece con 25 puntos de mundial porque ganó esa misma carrera. Para predecir una carrera usamos la fila de la carrera **anterior** de la misma temporada.

`constructor_results.csv` (puntos de cada equipo en cada carrera) tiene el mismo problema y se trata igual.

## `lap_times.csv` y `pit_stops.csv`

Tiempos de cada vuelta y cada parada en boxes. Todo ocurre durante la carrera. Para la carrera que se predice están prohibidos; de carreras anteriores se pueden usar, por ejemplo, para medir el ritmo reciente de un coche.

## `races.csv`: una fila por Gran Premio

| Columna | Significado | ¿Se puede usar? |
|---|---|---|
| `year`, `round` | temporada y número de carrera dentro de ella | Sí |
| `circuitId` | circuito | Sí |
| `date` | fecha | Sí, y sirve para ordenar en el tiempo |
| `name` | nombre del GP | Sí, aunque es redundante con el circuito |

## `status.csv`

Traduce `statusId` a texto: "Finished", "+1 Lap" (terminó doblado), "Engine", "Collision"… Para la carrera que se predice está prohibido, porque dice si el piloto terminó. De carreras anteriores lo usamos para medir la **fiabilidad**: cuántas veces se ha averiado el coche últimamente.

## Columnas que creamos en el notebook

| Columna | Qué es | ¿Entra en el modelo? |
|---|---|---|
| `race_idx` | número de orden cronológico de la carrera (0, 1, 2…) | No directamente; sirve para calcular "lo anterior" |
| `has_sprint` | si ese fin de semana hubo sprint | Sí |
| `podio` | 1 si `positionOrder ≤ 3` | No: es un objetivo |
| `top10` | 1 si `positionOrder ≤ 10` | No: es un objetivo |
| `terminado` | si terminó la carrera | No para esa carrera; sí su media en carreras anteriores |
| `abandono_mecanico` | si abandonó por avería | Igual que `terminado` |
| `abandono_accidente` | si abandonó por choque u otro motivo | Igual que `terminado` |
| `gap_pole_pct` | distancia a la pole en % | Sí |

## Variables del modelo (sección 5 del notebook)

Todas se pueden usar porque se conocen antes de la carrera: o son del fin de semana, o resumen carreras anteriores.

| Variable | Qué mide | Tipo |
|---|---|---|
| `grid_adj` | posición de salida (salir desde boxes = último + 1) | fin de semana |
| `pit_lane` | 1 si sale desde boxes | fin de semana |
| `quali_pos` | posición en la clasificación | fin de semana |
| `gap_pole_pct` | % más lento que la pole (máximo 10) | fin de semana |
| `penalizacion` | puestos perdidos entre clasificación y parrilla | fin de semana |
| `gap_companero` | `gap_pole_pct` propio menos el del compañero (negativo = más rápido) | fin de semana |
| `sprint_pos` | posición en la sprint (vacío si no hubo) | fin de semana |
| `has_sprint` | 1 si el fin de semana tuvo sprint | fin de semana |
| `n_coches` | coches en la carrera | fin de semana |
| `round` | número de carrera dentro de la temporada | fin de semana |
| `piloto_pos_5`, `piloto_pos_10` | posición media en las últimas 5 y 10 carreras | histórica |
| `piloto_puntos_5` | puntos medios en las últimas 5 | histórica |
| `piloto_top10_10`, `piloto_podio_10` | proporción de top 10 y de podios en las últimas 10 | histórica |
| `piloto_parrilla_5` | parrilla media en las últimas 5 | histórica |
| `piloto_remonta_10` | puestos ganados de media en carrera en las últimas 10 | histórica |
| `piloto_abandonos_10` | proporción de abandonos en las últimas 10 | histórica |
| `piloto_experiencia` | Grandes Premios disputados antes | histórica |
| `equipo_pos_5`, `equipo_puntos_5` | posición media y puntos por carrera del equipo en las últimas 5 | histórica |
| `equipo_averias_10` | proporción de coches del equipo con avería en las últimas 10 | histórica |
| `mundial_piloto_pts`, `_pos`, `_victorias` | mundial de pilotos antes de la carrera | histórica |
| `mundial_piloto_pct_lider` | puntos del piloto / puntos del líder | histórica |
| `mundial_equipo_pts`, `_pos` | mundial de constructores antes de la carrera | histórica |
| `companero_duelos_10` | proporción de carreras en que terminó delante de su compañero (últimas 10) | histórica |
| `piloto_circuito_pos` | posición media del piloto en sus últimas 5 visitas al circuito | histórica |
| `piloto_circuito_visitas` | carreras previas del piloto en el circuito | histórica |
| `circuito_spearman` | Spearman medio parrilla-resultado en las carreras anteriores del circuito | histórica |
| `rk_...` | puesto del piloto dentro de la carrera en una variable (1 = el mejor) | relativa |

## Resumen en una frase

Lo que se sabe el sábado (parrilla, clasificación, sprint, piloto, equipo, circuito) se puede usar para esa carrera. Lo que pasa el domingo (posición, puntos, vueltas, tiempos, abandono, mundial actualizado) solo se puede usar de carreras ya disputadas.
