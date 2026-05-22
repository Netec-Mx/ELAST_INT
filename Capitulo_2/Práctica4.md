# Práctica 4: Paginación de resultados

**Objetivo:** Implementar y comparar estrategias de paginación en
Elasticsearch 8.1, comprendiendo cómo afectan al **orden de resultados,
consistencia, consumo de recursos y comportamiento del \_score** en
distintos escenarios.

## Duración aproximada:
- 30 minutos.
  
**Actividad 1 — Paginación básica con from y size**

**Objetivo**

Obtener subconjuntos de resultados de una consulta.

**Actividad**

GET operations-\*/\_search  
{  
"from": 0,  
"size": 10,  
"query": {  
"match": {  
"Customers.name": "juan"  
}  
}  
}

**Explicación**

Los parámetros:

- from: desplazamiento (offset)

- size: número de resultados

El orden por defecto está determinado por \_score (de mayor a menor).
Esto implica que la paginación depende directamente de la relevancia
calculada.

Consideración importante:  
Elasticsearch debe calcular y ordenar todos los documentos hasta
alcanzar el rango solicitado (from + size). En páginas profundas (por
ejemplo, from: 10000), esto implica un costo elevado en memoria y CPU.

------------------------------------------------------------------------

**Actividad 2 — Impacto del \_score en la paginación**

**Objetivo**

Comprender cómo la relevancia afecta la estabilidad de los resultados
paginados.

**Actividad**

GET operations-\*/\_search  
{  
"from": 0,  
"size": 5,  
"query": {  
"match": {  
"Customers.name": "juan perez"  
}  
}  
}

Ejecutar nuevamente la misma consulta después de insertar nuevos
documentos.

**Explicación**

El \_score es dinámico y depende del contenido del índice. Si se agregan
documentos nuevos:

- El ranking puede cambiar

- Los documentos pueden moverse entre páginas

- Se puede producir inconsistencia en resultados paginados

Esto ocurre porque el cálculo de relevancia se reevalúa en cada
consulta.

------------------------------------------------------------------------

**Actividad 3 — Ordenamiento explícito con sort**

**Objetivo**

Estabilizar la paginación eliminando dependencia del \_score.

**Actividad**

GET operations-\*/\_search  
{  
"from": 0,  
"size": 10,  
"sort": \[  
{ "date": "desc" }  
\],  
"query": {  
"range": {  
"amount": { "gte": 1000 }  
}  
}  
}

**Explicación**

Al usar sort, el orden deja de depender del \_score. Esto tiene varias
implicaciones:

- El \_score ya no es el criterio principal

- La paginación se vuelve determinística

- Mejora la consistencia entre páginas

Este enfoque es recomendable en datos estructurados donde la relevancia
textual no es necesaria.

------------------------------------------------------------------------

**Actividad 4 — track_total_hits**

**Objetivo**

Controlar el costo de calcular el total de resultados.

**Actividad**

GET operations-\*/\_search  
{  
"track_total_hits": false,  
"size": 10,  
"query": {  
"match": {  
"Customers.name": "juan"  
}  
}  
}

**Explicación**

Por defecto, Elasticsearch calcula el número total de documentos
coincidentes. En grandes volúmenes, este cálculo puede ser costoso.

Opciones:

- true: calcula total exacto

- false: omite cálculo completo

- número: límite de precisión

Reducir este cálculo mejora el rendimiento sin afectar el \_score.

------------------------------------------------------------------------

**Actividad 5 — Limitaciones de from/size (deep pagination)**

**Objetivo**

Identificar problemas de escalabilidad.

**Actividad conceptual**

"from": 10000,  
"size": 10

**Explicación**

Este enfoque genera problemas porque:

- Elasticsearch debe procesar y ordenar muchos documentos

- El consumo de memoria crece linealmente

- Puede impactar la estabilidad del cluster

Además, el \_score debe calcularse para todos los documentos
considerados, lo que incrementa el costo computacional.

------------------------------------------------------------------------

**Actividad 6 — Paginación eficiente con search_after**

**Objetivo**

Implementar paginación escalable.

**Actividad**

**Primera consulta:**

GET operations-\*/\_search  
{  
"size": 5,  
"sort": \[  
{ "date": "desc" },  
{ "\_id": "asc" }  
\]  
}

**Siguiente página (usando último valor):**

GET operations-\*/\_search  
{  
"size": 5,  
"search_after": \["2026-04-29", "doc_id"\],  
"sort": \[  
{ "date": "desc" },  
{ "\_id": "asc" }  
\]  
}

**Explicación**

search_after evita el uso de offset (from) y permite:

- mejor rendimiento

- menor consumo de memoria

- escalabilidad en grandes volúmenes

Importante:  
El \_score no se usa como criterio principal en este tipo de paginación.
Se requiere un orden explícito y consistente.

------------------------------------------------------------------------

**Actividad 7 — Consistencia en paginación (consideración avanzada)**

**Objetivo**

Evitar inconsistencias en datos en tiempo real.

**Explicación**

En sistemas con alta tasa de escritura:

- nuevos documentos pueden alterar el orden

- el \_score puede cambiar entre consultas

- los resultados pueden duplicarse o perderse entre páginas

Solución conceptual:

- usar sort en lugar de \_score

- evitar paginación profunda

- considerar snapshots o estrategias de consistencia

------------------------------------------------------------------------

**Actividad 8 — Buenas prácticas**

**Recomendaciones**

- Evitar paginación profunda con from/size

- Usar search_after para grandes volúmenes

- Usar sort en lugar de \_score cuando no se requiera relevancia

- Reducir track_total_hits en consultas masivas

- Mantener consistencia en criterios de orden
