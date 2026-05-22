Objetivo: Diseñar e implementar una estrategia integral de gestión de
índices en Elasticsearch 8.1, basada en el uso de **index templates
(composable), component templates y buenas prácticas de modelado**, que
permita **automatizar la creación de índices de transacciones**,
garantizar **consistencia en mappings, settings y aliases**, y optimizar
el **rendimiento, escalabilidad y mantenibilidad** del sistema a partir
del modelo de datos

**1. Definición de Index Template para *Operations***

**Objetivo**

Establecer una plantilla que garantice consistencia en la creación de
índices de transacciones, asegurando que todos hereden la misma
configuración técnica y semántica.

**Actividad**

PUT \_index_template/operations_template  
{  
"index_patterns": \["operations-\*"\],  
"template": {  
"settings": {  
"number_of_shards": 1,  
"number_of_replicas": 1  
},  
"mappings": {  
"properties": {  
"custid": { "type": "keyword" },  
"amount": { "type": "double" },  
"date": { "type": "date" },  
"trans": { "type": "byte" },  
"branch": { "type": "integer" },  
"ip_trx": { "type": "ip" },  
"Customers": {  
"type": "nested",  
"properties": {  
"custid": { "type": "keyword" },  
"name": { "type": "text" },  
"lname": { "type": "text" }  
}  
},  
"Branchs": {  
"type": "nested",  
"properties": {  
"name": { "type": "text" },  
"ubication": { "type": "geo_point" }  
}  
}  
}  
},  
"aliases": {  
"transacciones_activas": {}  
}  
}  
}

**En donde:**

Un **index template** define reglas que se aplican automáticamente
cuando se crea un índice cuyo nombre coincide con un patrón. En este
caso:

- **settings**: controlan aspectos físicos (shards, réplicas).

- **mappings**: definen el esquema de datos y tipos.

- **aliases**: crean accesos lógicos desacoplados del nombre físico del
  índice.

El uso de nested es relevante porque permite preservar la relación entre
campos dentro de un mismo objeto, evitando inconsistencias en consultas
complejas.

------------------------------------------------------------------------

**2. Versionado de Templates**

**Objetivo**

Implementar control de cambios en configuraciones de índices.

**Actividad**

PUT \_index_template/operations_template  
{  
"version": 2,  
"index_patterns": \["operations-\*"\],  
"template": {  
"settings": {  
"number_of_shards": 2  
}  
}  
}

**En donde:**

El versionado permite:

- Controlar evolución del esquema sin pérdida de trazabilidad

- Coordinar cambios entre equipos

- Facilitar rollback en caso de errores

Es importante entender que el cambio **no impacta índices ya
existentes**, solo los nuevos.

------------------------------------------------------------------------

**3. Uso de Component Templates**

**Objetivo**

Separar configuraciones reutilizables para facilitar mantenimiento.

**Actividad**

**Template de configuración base:**

PUT \_component_template/base_settings  
{  
"template": {  
"settings": {  
"refresh_interval": "30s"  
}  
}  
}

**Template de mappings:**

PUT \_component_template/operations_mapping  
{  
"template": {  
"mappings": {  
"properties": {  
"custid": { "type": "keyword" }  
}  
}  
}  
}

**Template final:**

PUT \_index_template/operations_template_v2  
{  
"index_patterns": \["operations-\*"\],  
"priority": 200,  
"composed_of": \["base_settings", "operations_mapping"\]  
}

**En donde:**

- Los **component templates** permiten modularidad

- La propiedad **priority** resuelve conflictos cuando múltiples
  templates aplican al mismo índice

- Facilitan reutilización entre distintos dominios (logs, métricas,
  auditoría)

------------------------------------------------------------------------

**4. Automatización de índices diarios**

**Objetivo**

Implementar partición temporal para grandes volúmenes de datos.

**Actividad**

POST operations-2026-04-29/\_doc  
{  
"custid": "C001",  
"amount": 1500.50,  
"date": "2026-04-29",  
"trans": 1,  
"branch": 10,  
"ip_trx": "192.168.1.1"  
}

**En donde:**

Cuando un índice no existe, Elasticsearch lo crea automáticamente
aplicando el template correspondiente.

Este enfoque permite:

- Mejorar rendimiento de consultas

- Facilitar eliminación de datos antiguos

- Escalar horizontalmente

------------------------------------------------------------------------

**5. Configuración por entornos**

**Objetivo**

Adaptar configuración según necesidades operativas.

**Ejemplo**

**Desarrollo**

"number_of_replicas": 0

**Producción**

"number_of_replicas": 1,  
"refresh_interval": "30s"

**En donde:**

- En desarrollo se prioriza velocidad de indexación

- En producción se prioriza disponibilidad y consistencia

El parámetro refresh_interval impacta directamente la visibilidad de los
datos en búsquedas.

------------------------------------------------------------------------

**6. Resolución de conflictos y control de shards**

**Objetivo**

Evitar degradación del cluster por mala distribución de shards.

**Actividad**

PUT \_cluster/settings  
{  
"persistent": {  
"cluster.max_shards_per_node": 1000  
}  
}

**En donde:**

Un número excesivo de shards:

- Incrementa consumo de memoria

- Reduce eficiencia de búsquedas

- Aumenta latencia

Este límite actúa como mecanismo de protección.

------------------------------------------------------------------------

**7. Gestión de alias dinámicos**

**Objetivo**

Desacoplar lógica de consulta del almacenamiento físico.

**Actividad**

POST \_aliases  
{  
"actions": \[  
{ "add": { "index": "operations-2026-04-29", "alias":
"transacciones_activas" }},  
{ "remove": { "index": "operations-2026-04-28", "alias":
"transacciones_activas" }}  
\]  
}

**En donde:**

Los alias permiten:

- Cambiar índices sin modificar aplicaciones

- Implementar estrategias de rollover

- Facilitar mantenimiento

------------------------------------------------------------------------

**8. Optimización de mapeos**

**Objetivo**

Reducir uso de recursos y mejorar eficiencia de consultas.

**Actividad**

"amount": {  
"type": "scaled_float",  
"scaling_factor": 100  
}

"name": {  
"type": "text",  
"fields": {  
"keyword": { "type": "keyword" }  
}  
}

**En donde:**

- scaled_float reduce uso de almacenamiento

- multi-fields permite búsquedas y agregaciones sobre el mismo campo

------------------------------------------------------------------------

**9. Actividad adicional — Consulta sobre nested**

**Objetivo**

Consultar datos correctamente en estructuras anidadas.

**Ejemplo**

GET operations-\*/\_search  
{  
"query": {  
"nested": {  
"path": "Customers",  
"query": {  
"match": {  
"Customers.name": "Juan"  
}  
}  
}  
}  
}

**En donde:**

Sin nested, Elasticsearch podría mezclar datos de distintos objetos,
generando resultados incorrectos.

------------------------------------------------------------------------

**10. Actividad adicional — Búsqueda geoespacial**

**Objetivo**

Aprovechar el campo geo_point.

**Ejemplo**

GET operations-\*/\_search  
{  
"query": {  
"nested": {  
"path": "Branchs",  
"query": {  
"geo_distance": {  
"distance": "10km",  
"Branchs.ubication": {  
"lat": 19.4326,  
"lon": -99.1332  
}  
}  
}  
}  
}  
}

**En donde:**

Permite análisis como:

- transacciones cercanas

- detección de fraude geográfico

------------------------------------------------------------------------

**11. Actividad adicional — Buenas prácticas de modelado**

**Observaciones sobre tu modelo**

- accid como byte es limitado → usar integer

- nested debe usarse solo cuando hay necesidad real de consultas
  relacionales

- keyword debe usarse para identificadores

- text para búsquedas full-text

------------------------------------------------------------------------

**12. Actividad adicional — Simulación de carga**

**Objetivo**

Evaluar comportamiento del sistema.

**Ejemplo**

POST operations-\*/\_bulk  
{ "index": {} }  
{ "custid": "C002", "amount": 200, "date": "2026-04-29" }

**En donde:**

El uso de \_bulk mejora significativamente el rendimiento de indexación
frente a inserciones individuales.
