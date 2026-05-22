# Práctica 8: Aplicando agregaciones

**Objetivo del laboratorio**

Construir consultas analíticas mediante agregaciones y subagregaciones
que permitan **resumir, agrupar y calcular métricas** sobre datos
transaccionales, comprendiendo el impacto del tipo de campo, el uso de
keyword vs text, y la relación entre **query, filter y agregaciones**.

## Duración aproximada:
- 30 minutos.

**Actividad 1 — Aggregation básica (terms sobre keyword)**

**Objetivo**

Agrupar documentos por identificador de cliente.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"clientes": {  
"terms": {  
"field": "custid"  
}  
}  
}  
}

**En donde:**

Parámetros:

- size: 0: no devuelve documentos, solo agregaciones

- aggs: bloque de agregaciones

- terms: agrupa por valores únicos

- field: campo a agrupar (debe ser keyword o no analizado)

Resultado:

- lista de clientes

- conteo de documentos por cliente (doc_count)

------------------------------------------------------------------------

**Actividad 2 — Aggregation sobre campo text (uso correcto)**

**Objetivo**

Evitar errores al agrupar texto.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"tipos": {  
"terms": {  
"field": "descripcion.keyword"  
}  
}  
}  
}

**En donde:**

- Los campos text no pueden agregarse directamente

- .keyword usa la versión no analizada

Regla clave:  
Siempre usar keyword para agregaciones.

------------------------------------------------------------------------

**Actividad 3 — Métricas sobre campos numéricos**

**Objetivo**

Calcular estadísticas de montos.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"monto_promedio": {  
"avg": {  
"field": "amount"  
}  
},  
"monto_maximo": {  
"max": {  
"field": "amount"  
}  
},  
"monto_minimo": {  
"min": {  
"field": "amount"  
}  
},  
"total_transacciones": {  
"sum": {  
"field": "amount"  
}  
}  
}  
}

**En donde:**

Funciones:

- avg: promedio

- max: valor máximo

- min: valor mínimo

- sum: suma total

Requisito:

- Campo numérico (double, integer, etc.)

------------------------------------------------------------------------

**Actividad 4 — Subagregaciones (agrupación + métricas)**

**Objetivo**

Analizar montos por cliente.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"por_cliente": {  
"terms": {  
"field": "custid"  
},  
"aggs": {  
"monto_promedio": {  
"avg": {  
"field": "amount"  
}  
}  
}  
}  
}  
}

**En donde:**

- terms → agrupa

- aggs interno → calcula métricas por grupo

Resultado:

- cada cliente con su promedio

------------------------------------------------------------------------

**Actividad 5 — Aggregation por rango numérico**

**Objetivo**

Clasificar transacciones por monto.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"rangos_monto": {  
"range": {  
"field": "amount",  
"ranges": \[  
{ "to": 1000 },  
{ "from": 1000, "to": 5000 },  
{ "from": 5000 }  
\]  
}  
}  
}  
}

**En donde:**

Parámetros:

- ranges: define intervalos

Resultado:

- distribución de montos

------------------------------------------------------------------------

**Actividad 6 — Aggregation por fecha (date_histogram)**

**Objetivo**

Analizar transacciones en el tiempo.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"por_dia": {  
"date_histogram": {  
"field": "date",  
"calendar_interval": "day"  
}  
}  
}  
}

**En donde:**

Parámetros:

- date_histogram: agrupa por tiempo

- calendar_interval: intervalo (day, month, year)

Resultado:

- número de transacciones por día

------------------------------------------------------------------------

**Actividad 7 — Subagregación por fecha + métricas**

**Objetivo**

Ver evolución del monto.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"por_dia": {  
"date_histogram": {  
"field": "date",  
"calendar_interval": "day"  
},  
"aggs": {  
"monto_total": {  
"sum": {  
"field": "amount"  
}  
}  
}  
}  
}  
}

**En donde:**

- Combina tiempo + métricas

- Permite análisis tipo serie temporal

------------------------------------------------------------------------

**Actividad 8 — Aggregation con filtros**

**Objetivo**

Analizar subconjuntos de datos.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"transacciones_altas": {  
"filter": {  
"range": {  
"amount": { "gte": 1000 }  
}  
},  
"aggs": {  
"promedio": {  
"avg": {  
"field": "amount"  
}  
}  
}  
}  
}  
}

**En donde:**

- filter limita documentos

- no afecta \_score

- mejora rendimiento

------------------------------------------------------------------------

**Actividad 9 — Aggregation sobre nested**

**Objetivo**

Analizar datos anidados correctamente.

**Actividad**

GET operations-nested/\_search  
{  
"size": 0,  
"aggs": {  
"transacciones_nested": {  
"nested": {  
"path": "transacciones"  
},  
"aggs": {  
"por_tipo": {  
"terms": {  
"field": "transacciones.type"  
}  
}  
}  
}  
}  
}

**En donde:**

- nested: cambia el contexto

- path: campo nested

Sin esto, los resultados serían incorrectos.

------------------------------------------------------------------------

**Actividad 10 — Aggregation geoespacial**

**Objetivo**

Agrupar por ubicación.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"zonas": {  
"geohash_grid": {  
"field": "Branchs.ubication",  
"precision": 5  
}  
}  
}  
}

**En donde:**

Parámetros:

- geohash_grid: agrupa por zonas geográficas

- precision: nivel de detalle

Uso:

- análisis geográfico

- fraude

------------------------------------------------------------------------

**Actividad 11 — Aggregation por IP**

**Objetivo**

Detectar actividad por rango de IP.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"ip_ranges": {  
"ip_range": {  
"field": "ip_trx",  
"ranges": \[  
{ "to": "192.168.1.50" },  
{ "from": "192.168.1.50" }  
\]  
}  
}  
}  
}

**En donde:**

- ip_range: agrupa por rangos IP

- útil para seguridad

------------------------------------------------------------------------

**Actividad 12 — Pipeline aggregation**

**Objetivo**

Calcular métricas derivadas.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"por_cliente": {  
"terms": {  
"field": "custid"  
},  
"aggs": {  
"total": {  
"sum": {  
"field": "amount"  
}  
},  
"promedio_global": {  
"avg_bucket": {  
"buckets_path": "total"  
}  
}  
}  
}  
}  
}

**En donde:**

- avg_bucket: opera sobre resultados de otras agregaciones

- buckets_path: referencia agregación previa

------------------------------------------------------------------------

**Actividad 13 — Control de tamaño de buckets**

**Objetivo**

Evitar problemas de memoria.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"clientes": {  
"terms": {  
"field": "custid",  
"size": 10  
}  
}  
}  
}

**En donde:**

- size: limita número de buckets

- evita sobrecarga

------------------------------------------------------------------------

**Actividad 14 — Combinación completa**

**Objetivo**

Consulta analítica completa.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"query": {  
"range": {  
"date": {  
"gte": "now-7d/d"  
}  
}  
},  
"aggs": {  
"por_cliente": {  
"terms": {  
"field": "custid"  
},  
"aggs": {  
"por_tipo": {  
"terms": {  
"field": "trans"  
}  
},  
"monto_total": {  
"sum": {  
"field": "amount"  
}  
}  
}  
}  
}  
}

**En donde:**

- query: filtra documentos

- aggs: analiza resultados filtrados

Importante:

- Las agregaciones no dependen del \_score

- Operan sobre el conjunto de documentos resultante
