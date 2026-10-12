# Diccionario de features, NovaMarket semana 13

Catálogo de las 19 variables nuevas que se crean en `S13_ingenieria_de_caracteristicas.ipynb` sobre el
dataset integrado de la semana 12. El dataset resultante es `Datos/novamarket_con_features.csv`, con
620.000 filas y 146 columnas: las 127 originales más estas 19.

## Decisiones que aplican a todo el catálogo

**Fecha de corte: 16 de mayo de 2026.** Todas las variables de tiempo se calculan contra esta fecha fija,
no contra el día en que se corre el notebook. El dataset tiene pedidos continuos entre enero de 2025 y el
15 de mayo de 2026, y después un bloque disperso de 16.000 pedidos fechados a lo largo de 2027, sin nada
en el medio. El corte se fija el día siguiente al último pedido del periodo continuo. Los 16.000 pedidos
de 2027 conservan todas sus columnas en el dataset final pero no entran a calcular ninguna agregación.

**Base de agregación.** Las variables por cliente se calculan sobre 465.665 pedidos, los que cumplen tres
condiciones: pertenecen al conjunto de entrenamiento, tienen fecha registrada y son anteriores al corte.
Quedan cubiertos 147.353 de los 152.176 clientes del dataset.

**Unidad de análisis.** El dataset tiene una fila por pedido. Las variables de cliente describen al cliente
en la fecha de corte y se repiten en todos sus pedidos.

## Familia 1: variables derivadas a nivel de pedido

| Feature | Fórmula en palabras | Expresión | Tipo | Justificación de negocio |
|---|---|---|---|---|
| `dia_del_mes_pedido` | día calendario en que se hizo el pedido | `fecha_pedido.dt.day` | `Int64` (1 a 31) | El dataset ya tenía mes, trimestre y día de la semana, pero no la posición dentro del mes. En retail colombiano el ciclo de pago quincenal mueve la demanda, y sin esta columna el modelo no puede ver ese ciclo. |
| `es_quincena_pago` | verdadero si el pedido cae en los días de pago de quincena | `dia_del_mes_pedido in [14,15,16,29,30,31]` | `boolean` | Convierte la hipótesis de negocio en una sola variable: si el pico de ventas está pegado a los días de pago, conviene programar campañas y reposición de bodega alrededor de esas fechas. |
| `es_miembro_club_al_corte` | verdadero si el cliente ya estaba inscrito al club de fidelidad en la fecha de corte | `fecha_inscripcion_club < FECHA_CORTE` | `bool` | Separa tres situaciones que el CRM mezcla: cliente sin registro, cliente inscrito después del corte y miembro activo. Sin esta bandera, la antigüedad nula no se puede interpretar. |
| `antiguedad_club_dias` | días entre la inscripción al club y la fecha de corte | `(FECHA_CORTE - fecha_inscripcion_club).dt.days`, solo si era miembro | `float64` (1 a 1.231) | La antigüedad en el programa de fidelidad es el predictor más directo de permanencia que tiene marketing, y hoy solo existe como una fecha suelta que el modelo no puede usar. |

## Familia 2: agregaciones por cliente

| Feature | Fórmula en palabras | Expresión | Tipo | Justificación de negocio |
|---|---|---|---|---|
| `numero_compras_cliente` | cantidad de pedidos del cliente en la base | `groupby("id_cliente")["id_pedido"].count()` | `float64` (1 a 14) | Responde "¿cuántas veces nos ha comprado?" con un número. El dataset solo permitía contar filas a mano, cliente por cliente. |
| `total_gastado_cliente` | suma de los montos del cliente en la base | `groupby("id_cliente")["monto_compra"].sum()` | `float64` (0 a 13.751.000) | Es la medida de valor del cliente que pide el comité. Si todos los montos del cliente están sin registrar queda nulo, no cero: no gastó cero, no sabemos cuánto gastó. |
| `gasto_promedio_cliente` | monto total dividido por los pedidos con monto registrado | `groupby("id_cliente")["monto_compra"].mean()` | `float64` (0 a 4.438.500) | Distingue al cliente que gasta mucho porque compra seguido del que gasta mucho en cada compra. Son dos perfiles distintos y piden campañas distintas. |
| `categorias_distintas_cliente` | número de categorías diferentes que compró el cliente | `groupby("id_cliente")["categoria_producto"].nunique()` | `float64` (1 a 6) | Mide amplitud de canasta. Un cliente que ya compró en cuatro categorías es candidato a venta cruzada; uno concentrado en una sola es más frágil si esa categoría falla. |
| `compras_ultimos_90_dias` | pedidos del cliente en los 90 días previos al corte | conteo sobre `fecha_pedido >= FECHA_CORTE - 90 días` | `float64` (0 a 6) | Es la señal de actividad reciente que pide Camila para detectar fuga. El historial completo diluye la señal: un cliente con 10 compras hace dos años y ninguna este trimestre se ve igual de bueno que uno activo. |

## Familia 3: variables derivadas sobre las agregaciones

| Feature | Fórmula en palabras | Expresión | Tipo | Justificación de negocio |
|---|---|---|---|---|
| `dias_desde_ultima_compra` | días entre la última compra del cliente y la fecha de corte | `(FECHA_CORTE - fecha_ultima_compra).dt.days` | `float64` (1 a 500) | Es la pregunta original de Camila: "¿este cliente está activo o dormido?". Una fecha no responde nada por sí sola porque depende de contra qué se compare; este número sí se puede ordenar, filtrar y comparar entre clientes. |
| `es_cliente_recurrente` | verdadero si el cliente compró más de una vez | `numero_compras_cliente > 1` | `boolean` | La primera recompra es el umbral que marketing usa para decidir si un cliente entra a la base de retención o sigue en captación. |
| `puntos_por_mil_gastados` | puntos de fidelidad acumulados por cada mil pesos gastados | `puntos_fidelidad / (total_gastado / 1000)` | `float64` (0 a 555,8) | Cruza lo que el cliente recibió del programa con lo que realmente gastó. Un valor alto con gasto bajo apunta a puntos otorgados por fuera de la compra, y un valor muy bajo apunta a un cliente que gasta sin que el programa lo esté reconociendo. |
| `ticket_relativo_cliente` | monto del pedido dividido por el gasto promedio de su cliente | `monto_compra / gasto_promedio_cliente` | `float64` (0 a 126,9) | Responde si el pedido es grande o pequeño para ese cliente en particular, que es distinto de ser grande en términos absolutos. Un valor de 2 significa que gastó el doble de lo que suele gastar, y eso puede indicar una compra planeada, un regalo o un cambio de comportamiento. |

La división está protegida contra el cero: solo se calcula cuando el gasto en miles es mayor que cero, y en
los demás casos queda nulo. Sin esa condición un cliente con gasto cero produciría infinito.

**Sobre el rango de `ticket_relativo_cliente`.** El máximo es 126,9 y corresponde a clientes con un gasto
promedio muy bajo que hicieron un pedido grande. Es un valor real, no un error de cálculo, y en la semana
14 habrá que decidir si conviene recortarlo.

**Un candidato que se descartó.** La primera versión de esta variable dividía el monto del pedido por el
monto promedio de su categoría, con la idea de que 800.000 pesos no significan lo mismo en electrónica que
en papelería. En NovaMarket los promedios de las seis categorías difieren en 0,96 %, así que dividir por
ese promedio equivale a dividir por una constante: la correlación del resultado con `monto_compra` es
1,0000, es decir, era el mismo monto en otra escala. La versión contra el gasto del cliente correlaciona
0,5138 con `monto_compra`, que es la señal nueva que se buscaba. La comparación queda verificada en el
notebook.

## Familia 4: features de dominio

| Feature | Fórmula en palabras | Expresión | Tipo | Justificación de negocio |
|---|---|---|---|---|
| `rfm_score` | suma de los cuartiles del cliente en recencia, frecuencia y monto | `score_R + score_F + score_M` | `float64` (3 a 12) | Es el criterio con el que el equipo de marketing ya clasifica clientes a mano en una hoja de cálculo. Resume tres columnas en un solo número comparable entre clientes y hace calculable un criterio que hoy vive en la cabeza del equipo. |
| `segmento_rfm` | etiqueta de negocio por tramos de `rfm_score` | `pd.cut` en tramos 3-4, 5-6, 7-8, 9-10, 11-12 | `category` (Perdido, En riesgo, Regular, Leal, Campeon) | Traduce el puntaje al lenguaje con el que se toman decisiones. Un comité no actúa sobre "rfm_score igual a 5", actúa sobre "30.419 clientes en riesgo". |
| `categoria_favorita` | categoría con más pedidos del cliente, desempatando por la compra más reciente | `idxmax` ordenado por pedidos y fecha | `object` (6 valores) | Define el contenido de cualquier campaña dirigida. Sin esta variable hay que leer todos los pedidos del cliente para saber qué ofrecerle. |
| `canal_principal` | canal con más pedidos del cliente, mismo desempate | `idxmax` sobre las indicadoras de canal | `object` (app, marketplace, tienda, web) | Define por dónde contactar al cliente. El canal estaba repartido en cuatro columnas indicadoras desde la semana 9 y no existía como preferencia del cliente. |
| `es_temporada_alta` | verdadero si el pedido cae en noviembre o diciembre | `fecha_pedido.dt.month in [11,12]` | `boolean` | Permite comparar el comportamiento de un cliente dentro y fuera de temporada, y evita leer como crecimiento real lo que es solo el pico de fin de año. |

En las tres primeras se ve la regla de la familia: ninguna trae datos nuevos, todas combinan con criterio de
negocio variables que ya se calcularon en las familias anteriores. `rfm_score` depende de
`dias_desde_ultima_compra` (derivada), de `numero_compras_cliente` y de `total_gastado_cliente`
(agregaciones).

**Sobre el desempate de `categoria_favorita` y `canal_principal`.** El cliente promedio tiene 3,16 pedidos
repartidos en 2,49 categorías, así que lo más común es que dos categorías tengan un pedido cada una. Tomar
el máximo sin desempatar deja la decisión en el orden alfabético de las columnas y eso infla la primera
etiqueta: con ese criterio `belleza` salía con 43.081 clientes y `moda` con 13.098. Desempatando por la
compra más reciente, las seis categorías quedan entre 24.201 y 24.863 clientes.

**Sobre la escala invertida de la recencia.** En `score_R` las etiquetas van 4, 3, 2, 1 porque menos días
desde la última compra es mejor. En frecuencia y monto se aplica `rank(method="first")` antes de cortar en
cuartiles, porque hay muchos clientes empatados en el mismo número de compras y `qcut` no puede partir una
columna con bordes repetidos.

## Familia 5: variable de control

| Feature | Fórmula en palabras | Expresión | Tipo | Justificación de negocio |
|---|---|---|---|---|
| `sin_historial_cliente` | verdadero si el cliente no tiene ningún pedido en la base de agregación | `numero_compras_cliente.isna()` | `bool` | Marca de forma explícita a los 7.616 pedidos cuyas variables de cliente quedan nulas. Un hueco sin explicación en una tabla se interpreta como error de proceso; una columna que dice por qué está vacío se puede usar como filtro. |

## Cobertura y causas de los nulos

El criterio no es que no haya nulos, sino que cada nulo tenga una causa identificada. Hay cuatro causas y
ninguna columna nueva tiene valores infinitos.

| Causa | Pedidos afectados | Features afectadas |
|---|---|---|
| Pedido sin fecha registrada | 22.000 (3,55 %) | `dia_del_mes_pedido`, `es_quincena_pago`, `es_temporada_alta` |
| Pedido sin monto registrado, o cliente sin gasto promedio calculable | 89.976 (14,51 %) | `ticket_relativo_cliente` |
| Cliente sin registro en el CRM, o inscrito al club después del corte | 272.016 (43,87 %) | `antiguedad_club_dias`, y en parte `puntos_por_mil_gastados` (48.565, 7,83 %) |
| Cliente sin pedidos en la base de agregación | 7.616 (1,23 %) | todas las variables de cliente y las que se derivan de ellas |

A las variables que dependen del monto se suma un grupo más pequeño: 15.492 pedidos (2,50 %) de clientes
cuyos pedidos en la base no tienen ningún monto registrado, lo que afecta a `total_gastado_cliente`,
`gasto_promedio_cliente`, `rfm_score`, `segmento_rfm` y `ticket_relativo_cliente`.

## Control de fuga de información

Tres medidas, todas verificables en el notebook.

1. **Corte temporal.** Ninguna agregación usa pedidos posteriores al 16 de mayo de 2026. Los 16.000 pedidos
   fechados en 2027 quedan fuera de la base.
2. **Separación de conjuntos.** Las agregaciones y el monto promedio por categoría se calculan solo con
   pedidos del conjunto de entrenamiento. Ninguna estadística del conjunto de prueba entra en una variable
   que después alimente el modelo.
3. **Sin uso de la variable objetivo.** Ninguna de las 19 features se calcula a partir de `fuga_cliente`.

Queda una limitación que conviene declarar: las variables de cliente describen al cliente en la fecha de
corte y se repiten en todos sus pedidos, incluidos los anteriores al corte. Para un modelo que prediga a
nivel de pedido habría que recalcularlas con la fecha de cada pedido como referencia, lo que multiplica el
costo de cálculo. Para el objetivo de esta semana, que es evaluar al cliente en un momento fijo, la
definición actual es la correcta.

## Por qué ninguna de estas features repite una columna que ya existía

El dataset ya traía variables derivadas de la semana 10: `mes_pedido`, `trimestre_pedido`,
`dia_semana_pedido`, `es_fin_semana_pedido`, `longitud_comentario`, `num_palabras_comentario`,
`tiene_comentario`, `menciona_demora`, `grupo_edad`, `monto_compra_log` y `monto_compra_boxcox`. Ninguna de
las 19 nuevas las repite:

- las de calendario existentes describen mes, trimestre y día de la semana; `dia_del_mes_pedido` y
  `es_quincena_pago` describen la posición dentro del mes, que es un ciclo distinto
- `monto_compra_log` y `monto_compra_boxcox` son transformaciones de escala del mismo monto;
  `ticket_relativo_cliente` lo compara contra el gasto habitual del propio cliente, que es información que
  ninguna transformación del monto contiene
- `nivel_lealtad_ord` y `puntos_fidelidad_actuales` vienen declarados por el CRM;
  `puntos_por_mil_gastados` los pone en relación con el gasto observado en las ventas
- ninguna columna original resume el historial del cliente en un número, que es todo lo que aporta la
  familia de agregaciones

## Qué quedó fuera del catálogo

Los componentes intermedios `pedidos_con_monto`, `fecha_ultima_compra`, `puntos_fidelidad_cliente`,
`score_R`, `score_F` y `score_M` se calculan en el notebook pero no se suman al dataset final. No aportan
señal propia más allá de las variables que se construyeron con ellos, y mantenerlos duplicaría información
en una tabla que ya tiene 146 columnas.

El catálogo queda como un conjunto de candidatas. En la semana 14 el trabajo es el inverso: medir cuáles de
estas 19 aportan señal de verdad al problema de retención y descartar las demás.
