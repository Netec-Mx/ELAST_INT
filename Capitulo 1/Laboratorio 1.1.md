# Práctica 1: Importación de archivos de diferentes formatos

**Objetivos de aprendizaje**

- Comprender cómo se estructura un índice en Elasticsearch.

- Definir mappings explícitos y dinámicos para distintos tipos de datos.

- Crear y aplicar Index Templates y Component Templates.

- Ingerir datos desde un archivo CSV y validar búsquedas.

## Duración aproximada:
- 60 minutos.

**Escenario**

**Ejercicio 1: Analizar el dataset**

**Propósito:** identificar qué representa cada columna y cómo mapearla
en Elasticsearch.

- id → Identificador único de la operación → **keyword**

- customer → Nombre completo → **text**

- region, country, city → Ubicación geográfica → **keyword**

- ip → Dirección IP → **ip**

- operation_type → Tipo de operación (Loan, Transfer, etc.) →
  **keyword**

- amount → Valor numérico → **scaled_float**

- branch → Nombre de la sucursal → **keyword**

- executive → Responsable de la operación → **text**

- date → Fecha de la operación → **date**

**Explicación:** este análisis es equivalente a diseñar el esquema de
una base de datos relacional. En Elasticsearch lo llamamos *mapping*.

**Ejercicio 2: Crear un índice con mapping explícito**

**Propósito:** controlar cómo se almacenan los datos para evitar errores
y mejorar rendimiento.

json

PUT operations-v1

{

"settings": {

"number_of_shards": 1,

"number_of_replicas": 1

},

"mappings": {

"properties": {

"id": { "type": "keyword" },

"customer": { "type": "text" },

"region": { "type": "keyword" },

"country": { "type": "keyword" },

"city": { "type": "keyword" },

"ip": { "type": "ip" },

"operation_type": { "type": "keyword" },

"amount": { "type": "scaled_float", "scaling_factor": 100 },

"branch": { "type": "keyword" },

"executive": { "type": "text" },

"date": { "type": "date" }

}

}

}

**Explicación:** el mapping explícito asegura consistencia.
Elasticsearch no adivina tipos de datos, lo que es crítico en
producción.

**Ejercicio 3: Ingestar datos del CSV**

**Propósito:** cargar operaciones reales en el índice.

1.  Convertir las filas del CSV en documentos JSON.

2.  Usar la API Bulk para insertar múltiples documentos:

json

POST \_bulk

{ "index": { "\_index": "operations-v1", "\_id": "1499379" } }

{ "id": "1499379", "customer": "Dikova Plano Qaisar", "region": "South
America", "country": "Venezuela", "city": "Caracas", "ip":
"84.189.113.182", "operation_type": "Loan", "amount": 1378463.28,
"branch": "Oxxo", "executive": "Gabriel del Tomas", "date": "2026-06-24"
}

**Explicación:** cada documento representa una transacción financiera.
Elasticsearch lo indexa y lo hace buscable casi en tiempo real.

**Ejercicio 4: Validar búsquedas**

**Propósito:** comprobar que el mapping funciona.

- Buscar operaciones en Caracas:

json

GET operations-v1/\_search

{

"query": { "term": { "city": "Caracas" } }

}

- Buscar operaciones de tipo Loan mayores a 1,000,000:

json

GET operations-v1/\_search

{

"query": {

"bool": {

"must": \[

{ "term": { "operation_type": "Loan" } },

{ "range": { "amount": { "gte": 1000000 } } }

\]

}

}

}

**Explicación:** term busca coincidencias exactas, mientras que range
permite búsquedas por intervalos.

**Ejercicio 5: Crear un Index Template**

**Propósito:** automatizar configuraciones para futuros índices.

json

PUT \_index_template/operations_template

{

"index_patterns": \["operations-\*"\],

"priority": 200,

"template": {

"settings": { "number_of_shards": 1, "number_of_replicas": 1 },

"mappings": {

"properties": {

"id": { "type": "keyword" },

"customer": { "type": "text" },

"amount": { "type": "scaled_float", "scaling_factor": 100 },

"date": { "type": "date" }

}

},

"aliases": { "operations": {} }

}

}

**Explicación:** cualquier índice que empiece con operations- heredará
automáticamente este esquema.

**Ejercicio 6: Dynamic Templates**

**Propósito:** dar flexibilidad para campos desconocidos.

json

PUT operations-flex

{

"mappings": {

"dynamic_templates": \[

{

"strings_as_keyword": {

"match_mapping_type": "string",

"mapping": { "type": "keyword" }

}

}

\]

}

}

**Explicación:** cualquier campo nuevo de tipo string se almacenará como
keyword, evitando errores de análisis.
