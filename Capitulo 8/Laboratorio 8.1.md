# Práctica 14 : Consultas con Elasticsearch SQL

**Objetivo:** Construir consultas SQL en Elasticsearch para generar
reportes tabulares, comprendiendo:

- sintaxis SELECT

- ejecución vía REST

- control de acceso

- limitaciones y buenas prácticas

## Duración aproximada:
- 60 minutos.

------------------------------------------------------------------------

**8.1 Introducción a SQL en Elasticsearch**

**En donde:**

Elasticsearch SQL:

- permite usar sintaxis SQL estándar

- **no ejecuta SQL directamente**

- traduce la consulta a Query DSL internamente

Flujo interno:

1.  SQL → traducido a JSON (DSL)

2.  ejecución distribuida en shards

3.  resultado tabular

Esto significa:

- es ideal para reportes

- no reemplaza el motor nativo

------------------------------------------------------------------------

**Actividad 1 — Consulta básica tipo auditoría**

POST \_sql?format=txt  
{  
"query": "SELECT \* FROM operations-2026.04.29 LIMIT 5"  
}

**En donde:**

- SELECT \*: todos los campos

- FROM: índice (equivalente a tabla)

- LIMIT: controla volumen

Uso real:

- auditoría

- validación de datos

------------------------------------------------------------------------

**8.2 Estructura básica SELECT**

**Sintaxis general**

SELECT campo1, campo2  
FROM indice  
WHERE condicion  
ORDER BY campo DESC  
LIMIT N

(estructura basada en ANSI SQL )

------------------------------------------------------------------------

**Actividad 2 — Consulta filtrada**

POST \_sql?format=txt  
{  
"query": "  
SELECT custid, amount  
FROM operations-2026.04.29  
WHERE amount \> 1000  
ORDER BY amount DESC  
LIMIT 10  
"  
}

**En donde:**

- WHERE: filtro

- ORDER BY: ordenamiento

- LIMIT: control de resultados

------------------------------------------------------------------------

**Actividad 3 — Consulta sobre logs**

POST \_sql?format=txt  
{  
"query": "  
SELECT service, status, response_time_ms  
FROM logs-app-2026.04.29  
WHERE status \>= 400  
ORDER BY response_time_ms DESC  
"  
}

**En donde:**

- ideal para análisis de errores

- SQL facilita lectura frente a DSL

------------------------------------------------------------------------

**Búsqueda de texto (importante)**

POST \_sql?format=txt  
{  
"query": "  
SELECT message  
FROM logs-app-2026.04.29  
WHERE MATCH(message, 'error')  
"  
}

**En donde:**

- MATCH: equivalente a full-text search

- funciona sobre campos text

------------------------------------------------------------------------

**8.3 Control de acceso mediante roles**

**En donde:**

El control se basa en RBAC:

- roles → permisos

- usuarios → asignación de roles

- privilegios → acciones permitidas

El SQL respeta estos permisos automáticamente

**Actividad 4 — Crear rol de solo lectura**

PUT /\_security/role/operaciones_reader  
{  
"indices": \[  
{  
"names": \[ "operations-\*" \],  
"privileges": \[ "read", "view_index_metadata" \]  
}  
\]  
}

------------------------------------------------------------------------

**Actividad 5 — Crear usuario**

POST /\_security/user/analista  
{  
"password": "password_seguro",  
"roles": \[ "operaciones_reader" \]  
}

------------------------------------------------------------------------

**Actividad 6 — Validación**

POST \_sql?format=txt  
{  
"query": "SELECT \* FROM operations-2026.04.29"  
}

**En donde:**

- si el usuario no tiene permisos → error

- SQL hereda seguridad del cluster

------------------------------------------------------------------------

**8.4 Restricciones por campos e índices**

**En donde:**

Se aplican dos niveles:

**FLS (Field Level Security)**

- restringe columnas

**DLS (Document Level Security)**

- restringe filas

Estas reglas se aplican automáticamente en SQL

------------------------------------------------------------------------

**Actividad 7 — Restringir campos**

PUT /\_security/role/operaciones_limitadas  
{  
"indices": \[  
{  
"names": \[ "operations-\*" \],  
"privileges": \[ "read" \],  
"field_security": {  
"grant": \[ "custid", "amount" \]  
}  
}  
\]  
}

------------------------------------------------------------------------

**Actividad 8 — Restringir documentos**

PUT /\_security/role/operaciones_filtradas  
{  
"indices": \[  
{  
"names": \[ "operations-\*" \],  
"privileges": \[ "read" \],  
"query": {  
"range": {  
"amount": { "gte": 1000 }  
}  
}  
}  
\]  
}

------------------------------------------------------------------------

**En donde:**

- FLS → elimina columnas del resultado

- DLS → aplica filtro automático

------------------------------------------------------------------------

**8.5 Ejecución vía HTTP (REST API)**

**En donde:**

Endpoint principal:

POST /\_sql

Características:

- solo POST

- recibe JSON

- ejecuta consulta

------------------------------------------------------------------------

**Actividad 9 — Uso de parámetros**

POST \_sql?format=json  
{  
"query": "  
SELECT custid, amount  
FROM operations-2026.04.29  
WHERE amount \> 1000  
",  
"fetch_size": 5  
}

**En donde:**

- format: tipo de salida

- fetch_size: controla batch

------------------------------------------------------------------------

**Actividad 10 — Traducir SQL a DSL**

POST \_sql/translate  
{  
"query": "SELECT \* FROM operations-2026.04.29 WHERE amount \> 1000"  
}

**En donde:**

- permite ver ejecución real

- útil para optimización

------------------------------------------------------------------------

**8.6 Nomenclatura de objetos**

**En donde:**

Reglas clave:

- índices → minúsculas

- campos → snake_case

- evitar caracteres especiales

------------------------------------------------------------------------

**Actividad 11 — Uso correcto de nombres**

Correcto:

SELECT custid, amount FROM operations-2026.04.29

Incorrecto:

SELECT CustID FROM Operations

------------------------------------------------------------------------

**Caso especial — campos con punto**

SELECT "host.name" FROM logs-app-\*

**En donde:**

- puntos → estructura anidada

- deben ir entre comillas

------------------------------------------------------------------------

**8.7 Cuándo usar SQL vs DSL vs ES\|QL**

**Comparación**

Según el documento :

**SQL**

- reportes tabulares

- integración BI

**Query DSL**

- búsquedas complejas

- scoring avanzado

**ES\|QL**

- análisis de logs

- pipelines

------------------------------------------------------------------------

**Actividad 12 — Comparación práctica**

**SQL**

SELECT custid, SUM(amount)  
FROM operations-\*  
GROUP BY custid

------------------------------------------------------------------------

**ES\|QL equivalente**

FROM operations-\*  
\| STATS total = SUM(amount) BY custid

------------------------------------------------------------------------

**DSL equivalente**

{  
"aggs": {  
"por_cliente": {  
"terms": { "field": "custid" },  
"aggs": {  
"total": { "sum": { "field": "amount" } }  
}  
}  
}  
}

------------------------------------------------------------------------

**En donde:**

- SQL → más legible

- DSL → más potente

- ES\|QL → más analítico

------------------------------------------------------------------------

**Buenas prácticas**

1.  Usar SQL para reportes

2.  Evitar consultas muy complejas

3.  Validar con \_sql/translate

4.  Usar índices específicos

5.  Cuidar tipos de datos (keyword vs text)
