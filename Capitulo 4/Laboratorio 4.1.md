**Objetivo:** Diseñar analizadores personalizados en Elasticsearch para
controlar el proceso de análisis de texto (tokenización, normalización y
filtrado), evaluando su impacto en búsquedas, scoring y agregaciones.

**Actividad 1 — Comparación de analizadores integrados**

**Objetivo**

Comprender cómo Elasticsearch transforma texto en tokens usando
analizadores predefinidos.

**Actividad**

POST \_analyze  
{  
"analyzer": "standard",  
"text": "Transferencia Bancaria 2026"  
}

POST \_analyze  
{  
"analyzer": "whitespace",  
"text": "Transferencia Bancaria 2026"  
}

POST \_analyze  
{  
"analyzer": "keyword",  
"text": "Transferencia Bancaria 2026"  
}

**En donde:**

Parámetros:

- analyzer: define el analizador a utilizar

  - standard: divide por palabras, elimina puntuación, aplica minúsculas

  - whitespace: divide solo por espacios, no normaliza

  - keyword: no divide el texto, lo trata como un solo token

- text: contenido a analizar

Resultado esperado:

- standard → \["transferencia", "bancaria", "2026"\]

- whitespace → \["Transferencia", "Bancaria", "2026"\]

- keyword → \["Transferencia Bancaria 2026"\]

Conclusión técnica:  
El tipo de analizador define directamente cómo se construye el índice
invertido y, por tanto, cómo se calculará el \_score en búsquedas
posteriores.

------------------------------------------------------------------------

**Actividad 2 — Creación de analizador personalizado básico**

**Objetivo**

Construir un analizador que elimine ruido lingüístico.

**Actividad**

PUT operaciones-analizador  
{  
"settings": {  
"analysis": {  
"analyzer": {  
"custom_basic": {  
"type": "custom",  
"tokenizer": "standard",  
"filter": \[  
"lowercase",  
"stop"  
\]  
}  
}  
}  
}  
}

Validación:

POST operaciones-analizador/\_analyze  
{  
"analyzer": "custom_basic",  
"text": "Transferencia de dinero en la cuenta"  
}

**En donde:**

Parámetros clave:

- settings.analysis: sección donde se definen componentes de análisis

- analyzer.custom_basic: nombre del analizador

- type: custom: indica que se construye manualmente

- tokenizer: standard: divide el texto en palabras

- filter: lista de filtros aplicados a los tokens

Filtros:

- lowercase: convierte todo a minúsculas → evita duplicidad (Juan vs
  juan)

- stop: elimina palabras vacías como “de”, “la”, “en”

Impacto:

- Reduce ruido

- Mejora precisión de búsquedas

- Optimiza cálculo de \_score al eliminar términos irrelevantes

------------------------------------------------------------------------

**Actividad 3 — Analizador personalizado eliminando números**

**Objetivo**

Controlar qué tipo de información se indexa.

**Actividad**

PUT operaciones-analizador-numeros  
{  
"settings": {  
"analysis": {  
"filter": {  
"remove_numbers": {  
"type": "pattern_replace",  
"pattern": "\\d+",  
"replacement": ""  
}  
},  
"analyzer": {  
"custom_no_numbers": {  
"type": "custom",  
"tokenizer": "standard",  
"filter": \[  
"lowercase",  
"remove_numbers"  
\]  
}  
}  
}  
}  
}

Validación:

POST operaciones-analizador-numeros/\_analyze  
{  
"analyzer": "custom_no_numbers",  
"text": "Transferencia 12345 bancaria"  
}

**En donde:**

Parámetros:

- filter.remove_numbers: define un filtro personalizado

- type: pattern_replace: usa expresión regular

- pattern: "\\d+": detecta números

- replacement: "": elimina coincidencias

Resultado:

- Entrada: "Transferencia 12345 bancaria"

- Salida: \["transferencia", "bancaria"\]

Uso típico:

- eliminar IDs, códigos o ruido numérico

- mejorar calidad del índice para búsquedas textuales

------------------------------------------------------------------------

**Actividad 4 — Aplicación del analizador en mapping**

**Objetivo**

Integrar el analizador en un índice real.

**Actividad**

PUT operaciones-analyzer-mapping  
{  
"settings": {  
"analysis": {  
"analyzer": {  
"custom_text": {  
"type": "custom",  
"tokenizer": "standard",  
"filter": \["lowercase"\]  
}  
}  
}  
},  
"mappings": {  
"properties": {  
"descripcion": {  
"type": "text",  
"analyzer": "custom_text"  
}  
}  
}  
}

**En donde:**

Parámetros:

- mappings.properties.descripcion: campo a configurar

- type: text: habilita análisis

- analyzer: define qué analizador usar en indexación

Importante:

- El analizador se aplica al indexar datos

- Define cómo se almacenan los tokens en el índice

Impacto en \_score:

- Determina qué términos participan en la relevancia

- Afecta coincidencias parciales

------------------------------------------------------------------------

**Actividad 5 — Impacto del analizador en búsquedas**

**Objetivo**

Observar cómo cambia el matching.

**Actividad**

GET operaciones-analyzer-mapping/\_search  
{  
"query": {  
"match": {  
"descripcion": "transferencia"  
}  
}  
}

**En donde:**

Parámetros:

- match: usa el mismo proceso de análisis que el índice

- descripcion: campo analizado

Funcionamiento:

- La consulta se analiza igual que el documento

- Se comparan tokens

Impacto:

- Mejora recall (más resultados relevantes)

- Influye directamente en el \_score

------------------------------------------------------------------------

**Actividad 6 — Analizador con sinónimos**

**Objetivo**

Expandir semántica de búsqueda.

**Actividad**

PUT operaciones-sinonimos  
{  
"settings": {  
"analysis": {  
"filter": {  
"synonyms_filter": {  
"type": "synonym",  
"synonyms": \[  
"transferencia, envio",  
"retiro, extraccion"  
\]  
}  
},  
"analyzer": {  
"custom_synonyms": {  
"tokenizer": "standard",  
"filter": \[  
"lowercase",  
"synonyms_filter"  
\]  
}  
}  
}  
}  
}

**En donde:**

Parámetros:

- type: synonym: define equivalencias

- synonyms: lista de términos equivalentes

Impacto:

- Amplía coincidencias sin cambiar datos

- Mejora relevancia (\_score) al aumentar matches

------------------------------------------------------------------------

**Actividad 7 — Analizador y scoring**

**Objetivo**

Evaluar cómo el análisis afecta la relevancia.

**Actividad**

GET operaciones-analyzer-mapping/\_search  
{  
"query": {  
"match": {  
"descripcion": {  
"query": "transferencia internacional",  
"boost": 2  
}  
}  
}  
}

**En donde:**

Parámetros:

- boost: multiplica el \_score del campo

Relación con analyzer:

- Más tokens generados → más coincidencias posibles

- Mejor análisis → mejor ranking

------------------------------------------------------------------------

**Actividad 8 — Integración con bool, term y range**

**Objetivo**

Combinar análisis de texto con filtros estructurados.

**Actividad**

GET operations-\*/\_search  
{  
"query": {  
"bool": {  
"must": \[  
{  
"match": {  
"descripcion": "transferencia"  
}  
}  
\],  
"filter": \[  
{ "term": { "custid": "C001" } },  
{ "range": { "amount": { "gte": 1000 } } }  
\]  
}  
}  
}

**En donde:**

- must: participa en \_score

- filter: no afecta \_score

Relación con analyzer:

- El analyzer impacta solo en match

- Los filtros mantienen eficiencia

------------------------------------------------------------------------

**Actividad 9 — Uso con search_after**

**Objetivo**

Aplicar análisis en consultas eficientes.

**Actividad**

GET operations-\*/\_search  
{  
"size": 5,  
"sort": \[  
{ "date": "desc" },  
{ "\_id": "asc" }  
\],  
"query": {  
"match": {  
"descripcion": "transferencia"  
}  
}  
}

**En donde:**

- match usa analyzer

- sort evita dependencia del \_score

Permite:

- escalabilidad

- consistencia

------------------------------------------------------------------------

**Actividad 10 — Analizadores y agregaciones**

**Objetivo**

Evitar errores al agrupar texto.

**Actividad**

GET operations-\*/\_search  
{  
"size": 0,  
"aggs": {  
"tipos_transaccion": {  
"terms": {  
"field": "trans.keyword"  
}  
}  
}  
}

**En donde:**

Parámetros:

- terms: agrupa por valores únicos

- field.keyword: evita análisis

Regla clave:

- Nunca usar campos text en agregaciones

------------------------------------------------------------------------

**Actividad 11 — Subagregaciones**

**Objetivo**

Combinar análisis con métricas.

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

- terms: agrupación

- avg: métrica

El analyzer no interviene aquí directamente, pero define cómo se
indexaron los campos text usados en filtros previos.

------------------------------------------------------------------------

**Actividad 12 — Explain API**

**Objetivo**

Entender cómo el análisis afecta el score.

**Actividad**

GET operaciones-analyzer-mapping/\_explain/1  
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

- contribución de cada término

- cálculo exacto del \_score
