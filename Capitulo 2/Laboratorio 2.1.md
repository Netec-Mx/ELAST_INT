**Objetivo:** Desarrollar la capacidad de **gestionar documentos y
construir consultas eficientes en Elasticsearch 8.1**, comprendiendo el
comportamiento de los distintos tipos de búsqueda, la diferencia entre
datos analizados y exactos, y el impacto del **\_score en la relevancia
y el rendimiento**.

**Actividad 1 — Ciclo de vida del documento**

(2.1 Creación, búsqueda, actualización, eliminación)

**Objetivo**

Comprender cómo se almacenan y manipulan los documentos, base de
cualquier consulta.

**Actividad**

PUT operations-lab

POST operations-lab/\_doc/1  
{  
"custid": "C001",  
"amount": 1000,  
"date": "2026-04-29",  
"Customers": {  
"name": "Juan Perez"  
}  
}

GET operations-lab/\_search  
{  
"query": { "match_all": {} }  
}

POST operations-lab/\_update/1  
{  
"doc": { "amount": 1500 }  
}

DELETE operations-lab/\_doc/1

**En donde:**

El \_score no interviene en operaciones de escritura, pero sí en la
recuperación. El valor del \_score dependerá completamente de cómo se
indexaron los datos. Por ejemplo, si un campo se define como texto
analizado, el sistema podrá calcular relevancia; si es un campo exacto,
el \_score será menos significativo o uniforme. Por ello, el modelado
inicial condiciona la calidad del ranking.

------------------------------------------------------------------------

**Actividad 2 — Diferencia entre text vs keyword**

(2.2 tipos de datos y comportamiento de búsqueda)

**Objetivo**

Comprender cómo afecta el análisis de texto al cálculo del \_score.

**Actividad**

PUT operations-lab  
{  
"mappings": {  
"properties": {  
"name": {  
"type": "text",  
"fields": {  
"keyword": { "type": "keyword" }  
}  
}  
}  
}  
}

POST operations-lab/\_doc/1  
{  
"name": "Juan Perez"  
}

**Consulta tipo text (analizada):**

GET operations-lab/\_search  
{  
"query": {  
"match": {  
"name": "juan"  
}  
}  
}

**Consulta exacta (keyword):**

GET operations-lab/\_search  
{  
"query": {  
"term": {  
"name.keyword": "Juan Perez"  
}  
}  
}

**En donde:**

En campos text, Elasticsearch analiza el contenido dividiéndolo en
tokens. El \_score se calcula considerando coincidencias parciales,
frecuencia y relevancia lingüística. Por ejemplo, buscar “juan” puede
devolver “Juan Perez” con un score alto aunque no coincida exactamente.

En campos keyword, no hay análisis. El \_score no depende de similitud
sino de coincidencia exacta. Si el valor no coincide completamente, no
hay resultado. En estos casos, el \_score pierde relevancia práctica, ya
que todos los documentos coincidentes suelen tener valores similares.

------------------------------------------------------------------------

**Actividad 3 — Búsquedas por tipo de dato**

(2.2 aplicado al modelo)

**Objetivo**

Aplicar la query adecuada según el tipo de campo y entender su impacto
en el \_score.

**Actividad**

**Keyword (custid):**

{  
"term": {  
"custid": "C001"  
}  
}

**Numérico (amount):**

{  
"range": {  
"amount": {  
"gte": 1000,  
"lte": 5000  
}  
}  
}

**Fecha (date):**

{  
"range": {  
"date": {  
"gte": "now-1d/d",  
"time_zone": "-06:00"  
}  
}  
}

**IP:**

{  
"range": {  
"ip_trx": {  
"gte": "192.168.1.0",  
"lte": "192.168.1.255"  
}  
}  
}

**En donde:**

En búsquedas sobre datos estructurados (keyword, numéricos, fechas, IP),
el \_score no es el factor principal. Estas consultas se evalúan como
condiciones lógicas. Cuando se ejecutan en contexto de query, pueden
generar un \_score, pero este suele ser uniforme o poco significativo.
En la práctica, estas búsquedas deben moverse a contexto de filter para
evitar cálculos innecesarios y mejorar el rendimiento.

------------------------------------------------------------------------

**Actividad 4 — Diferencias entre match, term, range, bool**

(2.3)

**Objetivo**

Comprender cómo cada tipo de query influye en la relevancia.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"bool": {  
"must": \[  
{ "match": { "Customers.name": "juan" } }  
\],  
"filter": \[  
{ "term": { "custid": "C001" } },  
{ "range": { "amount": { "gte": 1000 } } }  
\]  
}  
}  
}

**En donde:**

El \_score se calcula únicamente con las cláusulas dentro de must o
should. En este caso:

- match genera \_score porque evalúa relevancia textual

- term y range en filter no afectan el \_score

Esto permite separar claramente:

- relevancia (qué tan bien coincide)

- restricción (qué documentos cumplen condiciones)

------------------------------------------------------------------------

**Actividad 5 — Query vs Filter**

(2.4 impacto en rendimiento y score)

**Objetivo**

Optimizar consultas eliminando cálculos innecesarios de relevancia.

**Actividad**

**Versión no optimizada:**

"must": \[  
{ "term": { "custid": "C001" } }  
\]

**Versión optimizada:**

"filter": \[  
{ "term": { "custid": "C001" } }  
\]

**En donde:**

En must, Elasticsearch calcula \_score incluso si no aporta valor. En
filter, la evaluación es binaria (cumple o no cumple), lo que elimina el
cálculo de relevancia. Esto reduce el consumo de CPU y mejora tiempos de
respuesta. En sistemas de alto tráfico, esta diferencia es crítica.

------------------------------------------------------------------------

**Actividad 6 — Análisis del \_score en texto**

(Profundización aplicada)

**Objetivo**

Observar cómo se calcula la relevancia.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"match": {  
"Customers.name": "juan perez"  
}  
}  
}

**En donde:**

El \_score se basa en varios factores:

- frecuencia del término dentro del documento

- rareza del término en el índice

- longitud del campo

- coincidencia de múltiples términos

Un documento que contenga “Juan Perez” exactamente tendrá mayor \_score
que uno que solo contenga “Juan”. Esto se debe a que coincide con más
términos y en mejor contexto.

------------------------------------------------------------------------

**Actividad 7 — Multi Search**

(2.5)

**Objetivo**

Ejecutar múltiples consultas optimizando latencia.

**Actividad**

POST \_msearch  
{"index": "operations-\*"}  
{"query": {"match": {"Customers.name": "juan"}}}  
  
{"index": "operations-\*"}  
{"query": {"range": {"amount": {"gte": 1000}}}}

**En donde:**

Cada consulta mantiene su propio \_score independiente. Esto permite
comparar resultados de distintas consultas sin interferencia. Es útil en
dashboards donde diferentes criterios requieren distintos tipos de
relevancia.

**Actividad 8 — Control de cláusulas bool**

(2.7)

**Objetivo**

Evitar degradación del rendimiento.

**Actividad**

**Ineficiente:**

"should": \[  
{"term": {"branch": 1}},  
{"term": {"branch": 2}},  
{"term": {"branch": 3}}  
\]

**Optimizado:**

"terms": {  
"branch": \[1,2,3\]  
}

**En donde:**

Muchas cláusulas generan múltiples evaluaciones de \_score. Agrupar
condiciones reduce el número de cálculos y mejora la eficiencia del
motor de búsqueda.

------------------------------------------------------------------------

**Actividad 9 — Optimización de respuesta**

(2.8)

**Objetivo**

Reducir volumen de datos retornados.

**Actividad**

GET operations-\*/\_search  
{  
"\_source": \["custid", "amount"\],  
"query": { "match_all": {} }  
}

**En donde:**

Reducir el tamaño de la respuesta no afecta el \_score, pero mejora
significativamente el rendimiento global del sistema. Menos datos
implican menor uso de red y menor latencia.

**Actividad 10 — Modificación del score (function_score)**

**Objetivo**

Controlar el ranking de resultados.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"function_score": {  
"query": {  
"match": { "Customers.name": "juan" }  
},  
"functions": \[  
{  
"field_value_factor": {  
"field": "amount",  
"factor": 1.5  
}  
}  
\]  
}  
}  
}

**En donde:**

El \_score original (basado en texto) se combina con factores numéricos.
En este caso, documentos con mayor monto obtendrán mayor relevancia.
Esto permite adaptar el ranking a necesidades del negocio.

------------------------------------------------------------------------

**Actividad 11 — script_score**

**Objetivo**

Aplicar lógica personalizada al ranking.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"script_score": {  
"query": { "match_all": {} },  
"script": {  
"source": "doc\['amount'\].value \* 2"  
}  
}  
}  
}

**En donde:**

El \_score se calcula completamente mediante un script. Esto permite
máxima flexibilidad, pero incrementa el costo computacional. Debe
utilizarse solo cuando las reglas de negocio no pueden expresarse con
mecanismos estándar.
