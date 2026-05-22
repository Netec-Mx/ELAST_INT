**Objetivo**

Identificar, reproducir y resolver los errores más comunes en
Elasticsearch, aplicando técnicas de diagnóstico y optimización para
mejorar la estabilidad y reducir latencias en consultas críticas (por
ejemplo, dashboards financieros en tiempo real).

------------------------------------------------------------------------

**Parte 1 — Identificación de errores comunes**

**Contexto técnico**

Como se describe en el material, Elasticsearch puede generar errores
debido a:

- conflictos de mapeo

- errores de sintaxis

- problemas de rendimiento

- inconsistencias en datos

Además, los errores suelen devolverse como **HTTP 400 con detalles en
JSON** .

------------------------------------------------------------------------

**Actividad 1 — Error de sintaxis (JSON mal formado)**

**Paso 1 — Ejecutar consulta incorrecta**

POST operations-\*/\_search  
{  
"query": {  
"match": {  
"custid": "C001"  
}  
  
}

**Resultado esperado**

Error tipo:

"error": "Failed to parse request body"

------------------------------------------------------------------------

**En donde:**

- Falta una llave de cierre }

- Elasticsearch valida estrictamente JSON

------------------------------------------------------------------------

**Corrección**

POST operations-\*/\_search  
{  
"query": {  
"match": {  
"custid": "C001"  
}  
}  
}

------------------------------------------------------------------------

**Actividad 2 — Error de mapeo (tipo incorrecto)**

**Paso 1 — Consulta incorrecta**

POST operations-\*/\_search  
{  
"query": {  
"match": {  
"amount": "alto"  
}  
}  
}

------------------------------------------------------------------------

**Resultado esperado**

Error o resultados incorrectos

------------------------------------------------------------------------

**En donde:**

- amount es double

- se está usando como texto

Este es un **conflicto de mapeo**, uno de los errores más comunes

------------------------------------------------------------------------

**Corrección**

POST operations-\*/\_search  
{  
"query": {  
"range": {  
"amount": {  
"gt": 1000  
}  
}  
}  
}

------------------------------------------------------------------------

**Actividad 3 — Error en campos text vs keyword**

**Consulta incorrecta**

POST customers/\_search  
{  
"query": {  
"term": {  
"name": "Juan"  
}  
}  
}

------------------------------------------------------------------------

**En donde:**

- name es text

- term requiere keyword

------------------------------------------------------------------------

**Corrección**

POST customers/\_search  
{  
"query": {  
"match": {  
"name": "Juan"  
}  
}  
}

------------------------------------------------------------------------

**Actividad 4 — Error en dashboards financieros**

**Escenario**

Dashboard en Kibana muestra datos incorrectos de transacciones.

**Problema**

Consulta:

{  
"query": {  
"term": {  
"amount": 1000  
}  
}  
}

------------------------------------------------------------------------

**En donde:**

- term busca coincidencia exacta

- valores double pueden no coincidir exactamente

------------------------------------------------------------------------

**Solución**

{  
"query": {  
"range": {  
"amount": {  
"gte": 1000,  
"lte": 1000  
}  
}  
}  
}

------------------------------------------------------------------------

**Parte 2 — Diagnóstico de errores**

**Concepto clave**

El diagnóstico se basa en:

- estado del clúster

- logs

- métricas de rendimiento

**Actividad 5 — Verificar estado del clúster**

GET \_cluster/health

------------------------------------------------------------------------

**Interpretación**

- green → OK

- yellow → replicas faltantes

- red → datos no disponibles

------------------------------------------------------------------------

**En donde:**

Un dashboard lento o con errores puede deberse a estado:

- amarillo → menor redundancia

- rojo → pérdida de datos

------------------------------------------------------------------------

**Actividad 6 — Diagnóstico de índices**

GET \_cat/indices?v

------------------------------------------------------------------------

**En donde:**

Permite detectar:

- índices no asignados

- shards problemáticos

------------------------------------------------------------------------

**Actividad 7 — Diagnóstico de nodos**

GET \_cat/nodes?v

------------------------------------------------------------------------

**En donde:**

- identifica nodos saturados

- útil en problemas de latencia

------------------------------------------------------------------------

**Actividad 8 — Validación de consultas**

POST operations-\*/\_validate/query  
{  
"query": {  
"match": {  
"custid": "C001"  
}  
}  
}

------------------------------------------------------------------------

**En donde:**

- valida sin ejecutar

- evita errores en producción

------------------------------------------------------------------------

**Parte 3 — Reducción de latencia en queries**

**Problema típico**

Dashboard financiero lento en tiempo real.

------------------------------------------------------------------------

**Actividad 9 — Query ineficiente**

POST operations-\*/\_search  
{  
"query": {  
"bool": {  
"must": \[  
{ "match": { "custid": "C001" }},  
{ "range": { "amount": { "gt": 1000 }}}  
\]  
}  
}  
}

------------------------------------------------------------------------

**Problema**

- must calcula scoring

- innecesario para filtros

------------------------------------------------------------------------

**Optimización**

POST operations-\*/\_search  
{  
"query": {  
"bool": {  
"filter": \[  
{ "term": { "custid": "C001" }},  
{ "range": { "amount": { "gt": 1000 }}}  
\]  
}  
}  
}

------------------------------------------------------------------------

**En donde:**

- filter:

  - no calcula score

  - usa caché

  - mejora rendimiento

------------------------------------------------------------------------

**Actividad 10 — Reducir tamaño de respuesta**

POST operations-\*/\_search  
{  
"\_source": \["custid", "amount"\],  
"query": {  
"match_all": {}  
}  
}

------------------------------------------------------------------------

**En donde:**

- evita cargar documentos completos

- reduce latencia

------------------------------------------------------------------------

**Actividad 11 — Evitar wildcard costoso**

Consulta incorrecta:

{  
"query": {  
"wildcard": {  
"name": "\*juan"  
}  
}  
}

------------------------------------------------------------------------

**Problema**

- wildcard al inicio es costoso

------------------------------------------------------------------------

**Alternativa**

- usar analyzer n-gram

- o búsqueda match

------------------------------------------------------------------------

**Parte 4 — Diagnóstico avanzado**

**Actividad 12 — Uso de logs en Kibana**

1.  Ir a Discover

2.  Filtrar:

level: ERROR

------------------------------------------------------------------------

**En donde:**

- permite detectar errores en tiempo real

- base para troubleshooting

------------------------------------------------------------------------

**Actividad 13 — Problemas de conexión**

**Error típico**

Backend closed connection

------------------------------------------------------------------------

**Solución**

POST \_reindex?wait_for_completion=false

**En donde:**

- evita timeout en procesos largos
