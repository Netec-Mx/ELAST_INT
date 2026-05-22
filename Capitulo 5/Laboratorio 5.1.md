# Práctica 9: Trabajando con painless Script

**Objetivo:** Aplicar Painless para implementar lógica de negocio
avanzada en Elasticsearch, comprendiendo su sintaxis, acceso a datos
(doc vs ctx), y su impacto en rendimiento y scoring.

## Duración aproximada:
- 60 minutos.

**Actividad 1 — Estructura básica de un script Painless**

**Objetivo**

Entender la sintaxis y tipos de datos.

**Actividad**

POST \_scripts/test_script  
{  
"script": {  
"lang": "painless",  
"source": """  
int x = 10;  
int y = 20;  
return x + y;  
"""  
}  
}

**En donde:**

Elementos clave:

- lang: define el lenguaje (painless)

- source: código del script

- return: obligatorio en contextos de cálculo

Sintaxis:

- Similar a Java (tipado fuerte)

- int, double, String, boolean

- def: tipo dinámico

Técnica:

- Preferir tipado estático → mejor rendimiento

- Usar def solo cuando el tipo es variable

------------------------------------------------------------------------

**Actividad 2 — Referencia a campos con doc**

**Objetivo**

Leer datos de forma eficiente.

**Actividad**

GET operations-\*/\_search  
{  
"script_fields": {  
"doble_monto": {  
"script": {  
"source": "doc\['amount'\].value \* 2"  
}  
}  
}  
}

**En donde:**

Parámetros:

- doc\['amount'\].value: acceso a doc_values

- solo lectura

- muy eficiente

Técnica:

- usar doc en búsquedas, filtros y scoring

- evitar ctx en este contexto

Nota clave (del material):  
doc_values son columnares y optimizados para lectura .

------------------------------------------------------------------------

**Actividad 3 — Referencia a campos con ctx**

**Objetivo**

Modificar documentos.

**Actividad**

POST operations-\*/\_update_by_query  
{  
"script": {  
"source": "ctx.\_source.flag = true"  
}  
}

**En donde:**

Parámetros:

- ctx.\_source: acceso al documento JSON

- mutable (permite escritura)

Técnica:

- usar solo cuando se necesita modificar

- es más lento porque deserializa JSON

------------------------------------------------------------------------

**Actividad 4 — Clasificación de transacciones**

**Objetivo**

Clasificar impacto: bajo, medio, alto.

**Actividad**

POST operations-\*/\_update_by_query  
{  
"script": {  
"source": """  
if (ctx.\_source.amount \< 1000) {  
ctx.\_source.impacto = 'bajo';  
} else if (ctx.\_source.amount \< 5000) {  
ctx.\_source.impacto = 'medio';  
} else {  
ctx.\_source.impacto = 'alto';  
}  
"""  
}  
}

**En donde:**

Técnicas utilizadas:

- if / else if / else → control de flujo

- asignación directa a \_source

Conceptos clave:

- lógica de negocio dentro del motor

- evita procesamiento externo

Optimización:

- evitar recalcular esto en queries → hacerlo en update

------------------------------------------------------------------------

**Actividad 5 — Crear campo riesgo_estimado**

**Objetivo**

Calcular riesgo basado en reglas.

**Actividad**

POST operations-\*/\_update_by_query  
{  
"script": {  
"source": """  
double monto = ctx.\_source.amount;  
int tipo = ctx.\_source.trans;  
  
if (monto \> 5000 && tipo == 2) {  
ctx.\_source.riesgo_estimado = 'alto';  
} else if (monto \> 2000) {  
ctx.\_source.riesgo_estimado = 'medio';  
} else {  
ctx.\_source.riesgo_estimado = 'bajo';  
}  
"""  
}  
}

**En donde:**

Parámetros:

- variables locales (double monto)

- operadores (&&, \>, ==)

Técnica:

- separar lógica en variables → mejora legibilidad

- usar tipado explícito para rendimiento

------------------------------------------------------------------------

**Actividad 6 — Uso de params (mejor práctica)**

**Objetivo**

Evitar recompilación de scripts.

**Actividad**

POST operations-\*/\_update_by_query  
{  
"script": {  
"source": """  
if (ctx.\_source.amount \> params.limite) {  
ctx.\_source.alerta = true;  
}  
""",  
"params": {  
"limite": 3000  
}  
}  
}

**En donde:**

Parámetros:

- params: variables externas

- evita recompilar scripts

Técnica crítica:

- nunca hardcodear valores

- mejora rendimiento y caching

------------------------------------------------------------------------

**Actividad 7 — Filtros con script**

**Objetivo**

Filtrar por lógica entre campos.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"bool": {  
"filter": {  
"script": {  
"script": {  
"source": "doc\['amount'\].value \> (doc\['trans'\].value \* 1000)"  
}  
}  
}  
}  
}  
}

**En donde:**

Técnicas:

- comparación entre campos

- uso de doc por eficiencia

Importante:

- estos filtros NO se cachean

- deben usarse solo cuando no hay alternativa

------------------------------------------------------------------------

**Actividad 8 — Scoring personalizado**

**Objetivo**

Modificar relevancia.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"function_score": {  
"query": {  
"match": { "descripcion": "transferencia" }  
},  
"script_score": {  
"script": {  
"source": "\_score \* doc\['amount'\].value"  
}  
}  
}  
}  
}

**En donde:**

Parámetros:

- \_score: score original

- doc\['amount'\]: factor adicional

Técnica:

- multiplicar score base

- integrar lógica de negocio

------------------------------------------------------------------------

**Actividad 9 — Script avanzado con logaritmos**

**Objetivo**

Evitar sesgos extremos.

**Actividad**

"source": "\_score \* Math.log1p(doc\['amount'\].value)"

**En donde:**

- Math.log1p(x) = log(1 + x)

- reduce impacto de valores muy grandes

Técnica:

- normalización matemática

- evita dominancia de outliers

------------------------------------------------------------------------

**Actividad 10 — Runtime field con Painless**

**Objetivo**

Calcular campo sin indexarlo.

**Actividad**

GET operations-\*/\_search  
{  
"runtime_mappings": {  
"impacto_calc": {  
"type": "keyword",  
"script": {  
"source": """  
if (doc\['amount'\].value \> 5000) emit('alto');  
else emit('bajo');  
"""  
}  
}  
}  
}

**En donde:**

Parámetros:

- emit(): devuelve valor

- runtime → cálculo en query

Técnica:

- no modificar índice

- útil para pruebas

------------------------------------------------------------------------

**Actividad 11 — Monitoreo de impacto**

**Objetivo**

Evaluar impacto en cluster.

**Actividad**

GET \_nodes/stats/script

**En donde:**

Permite analizar:

- compilaciones de scripts

- uso de CPU

- cache de scripts

Riesgos:

- exceso de scripts → saturación CPU

- límite de compilación (circuit breaker)

------------------------------------------------------------------------

**Actividad 12 — Optimización en operaciones masivas**

**Objetivo**

Evitar degradación de rendimiento.

**Técnicas clave**

1.  Usar ctx.op = 'noop'

if (ctx.\_source.amount \< 1000) {  
ctx.op = 'noop';  
}

Evita escritura innecesaria.

------------------------------------------------------------------------

2.  Usar batch_size en update_by_query

POST operations-\*/\_update_by_query?scroll_size=500

Reduce carga en cluster.

------------------------------------------------------------------------

3.  Filtrar antes de ejecutar script

"query": {  
"range": { "amount": { "gte": 1000 } }  
}

Menos documentos → menor costo.
