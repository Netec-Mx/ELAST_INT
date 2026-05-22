# Práctica 10: Actualizaciones masivas con Bulk API y Painless

**Objetivo:** Implementar actualizaciones masivas eficientes en
Elasticsearch utilizando **Bulk API + Painless**, comprendiendo:

## Duración aproximada:
- 60 minutos.
------------------------------------------------------------------------------------  
- formato NDJSON

- tipos de operación (index, update, delete)

- ejecución de scripts en lote

- optimización de rendimiento

- comportamiento interno (inmutabilidad y reindexación)

**Concepto clave previo (fundamental)**

Antes de los ejercicios, es importante entender:

- **Los documentos en Elasticsearch son inmutables**

- Una “actualización” en realidad:

  1.  crea un nuevo documento

  2.  marca el anterior como eliminado

Esto explica por qué las operaciones masivas deben optimizarse
cuidadosamente.

**Laboratorio 1 — Estructura básica de Bulk API**

**Actividad 1 — Inserción masiva con NDJSON**

POST \_bulk  
{ "index": { "\_index": "operations-2026.04.01", "\_id": "1" } }  
{ "custid": "C001", "amount": 1200, "trans": 1 }  
{ "index": { "\_index": "operations-2026.04.01", "\_id": "2" } }  
{ "custid": "C002", "amount": 5400, "trans": 2 }

**En donde:**

Estructura:

1.  Línea de acción

{ "index": { "\_index": "...", "\_id": "..." } }

2.  Línea de datos

Parámetros:

- \_index: índice destino

- \_id: identificador del documento

Proceso interno:

- Elasticsearch lee línea por línea

- distribuye cada operación al shard correspondiente

Ventaja clave:  
No necesita cargar todo el payload en memoria (streaming NDJSON).

------------------------------------------------------------------------

**Laboratorio 2 — Actualización parcial en Bulk**

**Actividad 2 — Uso de update + doc**

POST \_bulk  
{ "update": { "\_index": "operations-2026.04.01", "\_id": "1" } }  
{ "doc": { "amount": 1300 } }

**En donde:**

Parámetros:

- update: indica modificación parcial

- doc: contiene solo los campos a modificar

Proceso:

- Elasticsearch obtiene documento actual

- aplica cambios

- reindexa nueva versión

Diferencia clave:

- index → reemplaza todo

- update → modifica parcialmente

------------------------------------------------------------------------

**Laboratorio 3 — Bulk con Painless (lógica dinámica)**

**Actividad 3 — Incremento de monto**

POST \_bulk  
{ "update": { "\_index": "operations-2026.04.01", "\_id": "2" } }  
{  
"script": {  
"source": "ctx.\_source.amount += params.incremento",  
"params": {  
"incremento": 200  
}  
}  
}

**En donde:**

Parámetros:

- script.source: lógica en Painless

- ctx.\_source: documento actual

- params: variables externas

Técnica:

- evitar valores hardcodeados

- permite reutilización del script

Proceso:

1.  recupera documento

2.  ejecuta script

3.  genera nueva versión

------------------------------------------------------------------------

**Laboratorio 4 — Clasificación masiva con Painless**

**Actividad 4 — Clasificar impacto**

POST \_bulk  
{ "update": { "\_index": "operations-2026.04.01", "\_id": "1" } }  
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

- estructuras condicionales (if / else)

- acceso dinámico a campos

Uso real:

- clasificación de riesgo

- categorización de transacciones

------------------------------------------------------------------------

**Laboratorio 5 — Upsert en Bulk**

**Actividad 5 — Crear o actualizar**

POST \_bulk  
{ "update": { "\_index": "operations-2026.04.01", "\_id": "10" } }  
{  
"script": {  
"source": "ctx.\_source.amount += 1"  
},  
"upsert": {  
"amount": 1  
}  
}

**En donde:**

Parámetros:

- upsert: documento inicial si no existe

Proceso:

- existe → ejecuta script

- no existe → crea documento

Caso de uso:

- contadores

- métricas

**Laboratorio 6 — Eliminación masiva en Bulk**

**Actividad 6 — Delete**

POST \_bulk  
{ "delete": { "\_index": "operations-2026.04.01", "\_id": "2" } }

**En donde:**

- no requiere línea de datos

- marca documento como eliminado

------------------------------------------------------------------------

**Laboratorio 7 — Optimización con noop**

**Actividad 7 — Evitar reindexación innecesaria**

POST \_bulk  
{ "update": { "\_index": "operations-2026.04.01", "\_id": "1" } }  
{  
"script": {  
"source": """  
if (ctx.\_source.amount \> 1000) {  
ctx.\_source.flag = true;  
} else {  
ctx.op = 'noop';  
}  
"""  
}  
}

**En donde:**

Parámetro clave:

- ctx.op = 'noop'

Función:

- evita crear nueva versión del documento

- reduce CPU, disco y segmentación

------------------------------------------------------------------------

**Laboratorio 8 — Procesamiento masivo (batching)**

**Actividad 8 — Concepto de lotes**

Ejemplo conceptual:

- dividir en bloques de 500–1000 documentos

**En donde:**

Buenas prácticas:

- evitar payloads gigantes

- controlar memoria

- balancear throughput

------------------------------------------------------------------------

**Laboratorio 9 — Errores y control**

**Actividad 9 — Revisar respuesta de Bulk**

Ejecutar:

POST \_bulk  
...

**En donde:**

Respuesta contiene:

- errors: true/false

- detalle por operación

Técnica:

- validar errores parcialmente

- reintentar solo fallidos

------------------------------------------------------------------------

**Laboratorio 10 — Caso completo integrado**

**Actividad 10 — Clasificación + incremento + flag**

POST \_bulk  
{ "update": { "\_index": "operations-2026.04.01", "\_id": "1" } }  
{  
"script": {  
"source": """  
ctx.\_source.amount += params.incremento;  
  
if (ctx.\_source.amount \> 5000) {  
ctx.\_source.riesgo = 'alto';  
} else {  
ctx.\_source.riesgo = 'bajo';  
}  
""",  
"params": {  
"incremento": 100  
}  
}  
}

**En donde:**

Combina:

- cálculo dinámico

- clasificación

- uso de parámetros

------------------------------------------------------------------------

**Buenas prácticas clave**

1.  Usar NDJSON correctamente

2.  Evitar scripts complejos innecesarios

3.  Usar params siempre

4.  Usar noop cuando aplique

5.  Procesar en batches

6.  Validar errores de respuesta
