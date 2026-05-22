**Objetivo:** Diseñar consultas en Elasticsearch 8.1 que **aprovechen y
controlen el \_score**, combinando relevancia textual con reglas de
negocio (montos, fechas, tipo de transacción), y evaluando su impacto en
el ranking y el rendimiento.

**Actividad 1 — Observación básica del \_score con match**

**Objetivo**

Ver cómo se calcula el \_score en una búsqueda textual.

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

Parámetros:

- match: analiza el texto de la consulta con el mismo analyzer del campo

- "Customers.name": campo tipo text (analizado)

Funcionamiento:

- La consulta se tokeniza (ej. “juan”, “perez”)

- Elasticsearch calcula el \_score usando BM25 considerando:

  - frecuencia del término

  - rareza del término

  - longitud del campo

Resultado esperado:

- Documentos con ambos términos → mayor \_score

- Coincidencias parciales → menor \_score

------------------------------------------------------------------------

**Actividad 2 — Uso de boost en campos**

**Objetivo**

Modificar la importancia de un campo en el ranking.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"match": {  
"Customers.name": {  
"query": "juan",  
"boost": 2  
}  
}  
}  
}

**En donde:**

Parámetros:

- boost: factor multiplicador del \_score

Impacto:

- Duplica el peso del campo en el cálculo

- Aumenta la probabilidad de que estos documentos aparezcan primero

Uso típico:

- priorizar campos críticos (nombre, descripción, tipo de transacción)

------------------------------------------------------------------------

**Actividad 3 — Combinación con bool (must vs should)**

**Objetivo**

Controlar cómo múltiples condiciones afectan el \_score.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"bool": {  
"must": \[  
{ "match": { "Customers.name": "juan" } }  
\],  
"should": \[  
{ "match": { "descripcion": "transferencia" } }  
\]  
}  
}  
}

**En donde:**

Parámetros:

- must: condición obligatoria, contribuye al \_score

- should: opcional, aumenta el \_score si coincide

Funcionamiento:

- Todos los documentos deben cumplir must

- Los que además cumplan should reciben mayor \_score

Uso:

- ranking progresivo (más coincidencias → mejor posición)

------------------------------------------------------------------------

**Actividad 4 — Uso de filter para optimizar**

**Objetivo**

Separar relevancia de condiciones lógicas.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"bool": {  
"must": \[  
{ "match": { "descripcion": "transferencia" } }  
\],  
"filter": \[  
{ "term": { "custid": "C001" } },  
{ "range": { "amount": { "gte": 1000 } } }  
\]  
}  
}  
}

**En donde:**

Parámetros:

- filter: no afecta el \_score

Impacto:

- Reduce cálculos innecesarios

- Mantiene relevancia solo en texto

Resultado:

- Ranking basado en texto

- Filtro eficiente en datos estructurados

------------------------------------------------------------------------

**Actividad 5 — function_score (ajuste del ranking)**

**Objetivo**

Modificar el \_score usando valores numéricos.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"function_score": {  
"query": {  
"match": { "descripcion": "transferencia" }  
},  
"functions": \[  
{  
"field_value_factor": {  
"field": "amount",  
"factor": 1.2  
}  
}  
\]  
}  
}  
}

**En donde:**

Parámetros:

- function_score: permite modificar el \_score

- field_value_factor: usa valores de un campo numérico

- factor: multiplicador

Funcionamiento:

- Score base (texto) + ajuste por monto

Uso:

- priorizar transacciones grandes

- scoring financiero

------------------------------------------------------------------------

**Actividad 6 — control con weight**

**Objetivo**

Asignar peso fijo a condiciones.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"function_score": {  
"query": { "match_all": {} },  
"functions": \[  
{  
"filter": {  
"term": { "trans": 1 }  
},  
"weight": 3  
}  
\]  
}  
}  
}

**En donde:**

Parámetros:

- filter: condición

- weight: incremento fijo del \_score

Uso:

- priorizar tipos de transacción (ej. depósitos vs retiros)

------------------------------------------------------------------------

**Actividad 7 — script_score (lógica personalizada)**

**Objetivo**

Definir completamente el cálculo del \_score.

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

Parámetros:

- script: lógica en Painless

- doc\['amount'\].value: acceso al campo

Impacto:

- Control total del ranking

- Mayor costo computacional

Uso:

- modelos de fraude

- scoring financiero avanzado

------------------------------------------------------------------------

**Actividad 8 — combinación avanzada (texto + negocio)**

**Objetivo**

Crear una consulta realista.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"function_score": {  
"query": {  
"bool": {  
"must": \[  
{ "match": { "descripcion": "transferencia" } }  
\],  
"filter": \[  
{ "range": { "date": { "gte": "now-7d/d" } } }  
\]  
}  
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

- Texto define relevancia base

- Filtros reducen universo

- Factores ajustan ranking

Resultado:

- documentos recientes

- priorización por monto

- relevancia textual

------------------------------------------------------------------------

**Actividad 9 — uso de min_score**

**Objetivo**

Filtrar resultados poco relevantes.

**Actividad**

GET operations-\*/\_search  
{  
"min_score": 1.5,  
"query": {  
"match": {  
"descripcion": "transferencia"  
}  
}  
}

**En donde:**

Parámetros:

- min_score: umbral mínimo

Impacto:

- elimina ruido

- mejora calidad de resultados

------------------------------------------------------------------------

**Actividad 10 — interacción con sort**

**Objetivo**

Evaluar conflicto entre score y ordenamiento.

**Actividad**

GET operations-\*/\_search  
{  
"sort": \[  
{ "amount": "desc" }  
\],  
"query": {  
"match": {  
"descripcion": "transferencia"  
}  
}  
}

**En donde:**

- sort anula el uso del \_score como orden principal

- El \_score sigue calculándose pero no ordena resultados

Uso:

- reportes

- dashboards

------------------------------------------------------------------------

**Actividad 11 — explain para análisis de score**

**Objetivo**

Entender cómo se calcula el \_score.

**Actividad**

GET operations-\*/\_explain/1  
{  
"query": {  
"match": {  
"descripcion": "transferencia"  
}  
}  
}

**En donde:**

Permite ver:

- tokens generados

- peso de cada término

- cálculo detallado del \_score

------------------------------------------------------------------------

**Actividad 12 — buenas prácticas de scoring**

**Observaciones integradas**

- Usar match solo en campos text

- Usar filter para datos estructurados

- Evitar script_score en alto volumen

- Ajustar relevancia con boost antes de usar scripts

- Usar function_score como enfoque intermedio
