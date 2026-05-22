# Práctica 11: Aplicando Reindex API

**Objetivo:** Aplicar la API de **reindexación en Elasticsearch 8.1**
para:

## Duración aproximada:
- 30 minutos.

----------------------------------------------------------------------------------------------

- migrar datos entre índices

- transformar documentos durante la copia

- limpiar y enriquecer información

- optimizar estructuras de datos

Comprendiendo su proceso interno y su impacto en el rendimiento.

------------------------------------------------------------------------

**Concepto clave previo**

La Reindex API:

- **lee documentos desde un índice origen**

- **ejecuta opcionalmente transformaciones**

- **indexa en un índice destino**

Internamente:

1.  realiza una búsqueda (scroll)

2.  procesa documentos en lotes

3.  reindexa cada documento

Esto implica:

- consumo de CPU

- uso de red interna

- escritura intensiva en disco

------------------------------------------------------------------------

**Laboratorio 1 — Reindex básico**

**Actividad 1 — Copia de índice completo**

POST \_reindex  
{  
"source": {  
"index": "operations-2026.04.01"  
},  
"dest": {  
"index": "operations-backup"  
}  
}

**En donde:**

Parámetros:

- source.index: índice origen

- dest.index: índice destino

Proceso:

- lee todos los documentos

- los escribe en el nuevo índice

Uso:

- backup

- pruebas

- duplicación de datos

------------------------------------------------------------------------

**Laboratorio 2 — Reindex con transformación (Painless)**

**Actividad 2 — Clasificación de riesgo**

POST \_reindex  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-v2"  
},  
"script": {  
"source": """  
if (ctx.\_source.amount \> 5000) {  
ctx.\_source.riesgo = 'alto';  
} else {  
ctx.\_source.riesgo = 'bajo';  
}  
"""  
}  
}

**En donde:**

Parámetros:

- script: transformación por documento

- ctx.\_source: documento actual

Técnica:

- enriquecer datos durante migración

- evitar reprocesamiento posterior

------------------------------------------------------------------------

**Laboratorio 3 — Reindex con filtro**

**Actividad 3 — Migrar solo transacciones relevantes**

POST \_reindex  
{  
"source": {  
"index": "operations-\*",  
"query": {  
"range": {  
"amount": {  
"gte": 1000  
}  
}  
}  
},  
"dest": {  
"index": "operations-high"  
}  
}

**En donde:**

Parámetros:

- query: filtra documentos origen

Proceso:

- reduce volumen de datos

- mejora rendimiento

Uso:

- archivado selectivo

- segmentación de datos

------------------------------------------------------------------------

**Laboratorio 4 — Limpieza de datos**

**Actividad 4 — Eliminar campo innecesario**

POST \_reindex  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-clean"  
},  
"script": {  
"source": "ctx.\_source.remove('ip_trx')"  
}  
}

**En donde:**

- remove(): elimina campo

- útil para reducción de tamaño

Impacto:

- menor uso de almacenamiento

- mejor rendimiento en queries

------------------------------------------------------------------------

**Laboratorio 5 — Transformación estructural**

**Actividad 5 — Normalización de datos**

POST \_reindex  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-normalized"  
},  
"script": {  
"source": """  
ctx.\_source.custid = ctx.\_source.custid.toUpperCase();  
ctx.\_source.fecha_migracion = '2026-04-29';  
"""  
}  
}

**En donde:**

Técnicas:

- transformación de texto

- enriquecimiento con nuevos campos

Uso:

- estandarización

- auditoría

------------------------------------------------------------------------

**Laboratorio 6 — Validación con noop**

**Actividad 6 — Evitar documentos inválidos**

POST \_reindex  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-valid"  
},  
"script": {  
"source": """  
if (ctx.\_source.custid == null) {  
ctx.op = 'noop';  
}  
"""  
}  
}

**En donde:**

Parámetro clave:

- ctx.op = 'noop'

Función:

- evita indexar documentos inválidos

- mejora calidad de datos

**Laboratorio 7 — Reindex con renombrado de campos**

**Actividad 7 — Cambiar nombre de campo**

POST \_reindex  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-renamed"  
},  
"script": {  
"source": """  
ctx.\_source.monto = ctx.\_source.amount;  
ctx.\_source.remove('amount');  
"""  
}  
}

**En donde:**

- permite cambios de esquema

- útil en migraciones

------------------------------------------------------------------------

**Laboratorio 8 — Ejecución asíncrona**

**Actividad 8 — Evitar timeout**

POST \_reindex?wait_for_completion=false  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-large"  
}  
}

**En donde:**

- no bloquea la petición HTTP

- devuelve task_id

------------------------------------------------------------------------

**Laboratorio 9 — Monitoreo de tareas**

**Actividad 9 — Seguimiento**

GET \_tasks/\<task_id\>

**En donde:**

Permite ver:

- progreso

- documentos procesados

- errores

------------------------------------------------------------------------

**Laboratorio 10 — Paralelismo con slices**

**Actividad 10 — Optimización**

POST \_reindex?slices=auto  
{  
"source": {  
"index": "operations-\*"  
},  
"dest": {  
"index": "operations-parallel"  
}  
}

**En donde:**

Parámetros:

- slices: divide trabajo en paralelo

Uso:

- índices grandes

- clusters con múltiples shards

------------------------------------------------------------------------

**Laboratorio 11 — Uso de ingest pipeline**

**Actividad 11 — Pipeline durante reindex**

POST \_reindex  
{  
"source": {  
"index": "operations-raw"  
},  
"dest": {  
"index": "operations-clean",  
"pipeline": "pipeline_normalizacion"  
}  
}

**En donde:**

- delega lógica a pipeline

- mejora mantenibilidad

------------------------------------------------------------------------

**Laboratorio 12 — Caso completo integrado**

**Actividad 12 — Migración + limpieza + clasificación**

POST \_reindex  
{  
"source": {  
"index": "operations-\*",  
"query": {  
"range": {  
"date": {  
"gte": "now-30d/d"  
}  
}  
}  
},  
"dest": {  
"index": "operations-final"  
},  
"script": {  
"source": """  
if (ctx.\_source.amount \> 5000) {  
ctx.\_source.riesgo = 'alto';  
} else {  
ctx.\_source.riesgo = 'bajo';  
}  
  
ctx.\_source.remove('ip_trx');  
ctx.\_source.fecha_migracion = '2026-04-29';  
"""  
}  
}

**En donde:**

Integra:

- filtro por fecha

- transformación

- limpieza

- enriquecimiento

------------------------------------------------------------------------

**Buenas prácticas**

1.  Crear índice destino previamente con mappings correctos

2.  Filtrar datos antes de reindexar

3.  Usar scripts simples

4.  Usar noop para evitar basura

5.  Ejecutar en modo asíncrono en grandes volúmenes

6.  Monitorear tareas
