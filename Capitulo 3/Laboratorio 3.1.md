# Práctica 5 : Trabajando con jerarquías en documentos

**Objetivo:** Diseñar e implementar modelos de datos jerárquicos en
Elasticsearch 8.1, comprendiendo las diferencias con modelos
relacionales, evaluando el uso de **arreglos, documentos anidados
(nested) y relaciones tipo join**, y aplicando consultas eficientes que
equilibren **precisión, relevancia (\_score) y rendimiento**.

## Duración aproximada:
- 60 minutos.
  
**Actividad 1 — Necesidad de jerarquías vs modelo relacional**

**Objetivo**

Comprender por qué Elasticsearch no utiliza joins tradicionales.

**Actividad conceptual**

Modelo relacional típico:

- Cuenta → múltiples clientes

- Cuenta → múltiples transacciones

En un modelo relacional, esto implicaría varias tablas y joins.

En Elasticsearch, se modela como documento:

{  
"accid": 1,  
"acctype": 2,  
"titulares": \["C001", "C002"\],  
"transacciones": \[  
{ "amount": 1000, "type": 1 },  
{ "amount": 2000, "type": 2 }  
\]  
}

**En donde:**

Elasticsearch está optimizado para lectura distribuida, no para joins
complejos.

- Los datos se **desnormalizan**

- Se prioriza acceso rápido sobre consistencia relacional

- El \_score se calcula sobre documentos completos, no sobre
  combinaciones entre tablas

Esto implica que el diseño del documento es crítico para el rendimiento.

**Actividad 2 — Uso de arreglos (arrays)**

**Objetivo**

Evaluar cuándo usar arreglos simples.

**Actividad**

PUT accounts-lab  
{  
"mappings": {  
"properties": {  
"titulares": { "type": "keyword" }  
}  
}  
}

POST accounts-lab/\_doc  
{  
"accid": 1,  
"titulares": \["C001", "C002"\]  
}

Consulta:

GET accounts-lab/\_search  
{  
"query": {  
"term": {  
"titulares": "C001"  
}  
}  
}

**En donde:**

Los arrays en Elasticsearch:

- No tienen tipo especial, son listas del mismo tipo

- Funcionan bien para valores independientes

Limitación importante:  
Cuando los arrays contienen objetos complejos, Elasticsearch puede
mezclar valores entre elementos, generando resultados incorrectos.

Ejemplo de problema:

"transacciones": \[  
{ "tipo": "deposito", "monto": 100 },  
{ "tipo": "retiro", "monto": 500 }  
\]

Una consulta podría combinar "deposito" con 500, lo cual es incorrecto.
Aquí es donde se requiere nested.

------------------------------------------------------------------------

**Actividad 3 — Modelado con nested**

**Objetivo**

Mantener integridad de relaciones dentro de un documento.

**Actividad**

PUT operations-nested  
{  
"mappings": {  
"properties": {  
"transacciones": {  
"type": "nested",  
"properties": {  
"amount": { "type": "double" },  
"type": { "type": "keyword" },  
"date": { "type": "date" }  
}  
}  
}  
}  
}

POST operations-nested/\_doc  
{  
"accid": 1,  
"transacciones": \[  
{ "amount": 1000, "type": "deposito", "date": "2026-04-29" },  
{ "amount": 500, "type": "retiro", "date": "2026-04-30" }  
\]  
}

Consulta:

GET operations-nested/\_search  
{  
"query": {  
"nested": {  
"path": "transacciones",  
"query": {  
"bool": {  
"must": \[  
{ "term": { "transacciones.type": "deposito" } },  
{ "range": { "transacciones.amount": { "gte": 1000 } } }  
\]  
}  
}  
}  
}  
}

**En donde:**

El tipo nested:

- Indexa cada objeto como un documento interno independiente

- Mantiene la relación entre campos del mismo objeto

Impacto en \_score:

- Cada nested genera su propio cálculo de relevancia

- El score final se combina con el documento principal

Costo:

- Mayor uso de almacenamiento

- Queries más complejas

------------------------------------------------------------------------

**Actividad 4 — Consultas por tipo, fecha y cuenta**

**Objetivo**

Ejecutar consultas reales sobre transacciones.

**Actividad**

GET operations-nested/\_search  
{  
"query": {  
"bool": {  
"must": \[  
{  
"nested": {  
"path": "transacciones",  
"query": {  
"range": {  
"transacciones.date": {  
"gte": "now-7d/d"  
}  
}  
}  
}  
}  
\],  
"filter": \[  
{ "term": { "accid": 1 } }  
\]  
}  
}  
}

**En donde:**

- nested mantiene integridad de datos

- filter evita cálculo de \_score innecesario

- El \_score se basa solo en condiciones relevantes

------------------------------------------------------------------------

**Actividad 5 — Uso de Join (parent-child)**

**Objetivo**

Modelar relaciones sin duplicar datos.

**Actividad**

PUT accounts-join  
{  
"mappings": {  
"properties": {  
"relation": {  
"type": "join",  
"relations": {  
"account": "transaction"  
}  
}  
}  
}  
}

Insertar padre:

POST accounts-join/\_doc/1  
{  
"relation": "account"  
}

Insertar hijo:

POST accounts-join/\_doc/2?routing=1  
{  
"relation": {  
"name": "transaction",  
"parent": "1"  
},  
"amount": 1000  
}

Consulta:

GET accounts-join/\_search  
{  
"query": {  
"has_child": {  
"type": "transaction",  
"query": {  
"range": {  
"amount": { "gte": 1000 }  
}  
}  
}  
}  
}

**En donde:**

El modelo join:

- Evita duplicación de datos

- Permite relaciones dinámicas

Desventajas:

- Mayor costo en consultas

- Dependencia de routing

- Impacto negativo en rendimiento

El \_score puede combinarse entre padre e hijo, aumentando complejidad.

------------------------------------------------------------------------

**Actividad 6 — Comparación de enfoques**

| Técnica | Ventajas          | Desventajas          | Uso recomendado       |
|---------|-------------------|----------------------|-----------------------|
| Arrays  | Simple, eficiente | Problemas en objetos | Datos independientes  |
| Nested  | Precisión alta    | Mayor costo          | Relaciones internas   |
| Join    | Flexible          | Bajo rendimiento     | Casos muy específicos |

------------------------------------------------------------------------

**Actividad 7 — Control de cláusulas bool**

**Objetivo**

Optimizar queries complejas.

**Actividad**

{  
"query": {  
"bool": {  
"must": \[  
{ "match": { "Customers.name": "juan" } }  
\],  
"filter": \[  
{ "term": { "accid": 1 } },  
{ "range": { "amount": { "gte": 1000 } } }  
\]  
}  
}  
}

**En donde:**

Separar condiciones permite:

- reducir cálculos de \_score

- mejorar rendimiento

- mantener claridad lógica

------------------------------------------------------------------------

**Actividad 8 — Estrategias de optimización**

**Objetivo**

Reducir costo de consultas.

**Actividad**

GET operations-\*/\_search  
{  
"\_source": \["accid", "amount"\],  
"query": {  
"bool": {  
"filter": \[  
{ "term": { "accid": 1 } }  
\]  
}  
}  
}

**En donde:**

- Reducir \_source disminuye tamaño de respuesta

- Usar filter elimina cálculo de \_score

- Mejora latencia y uso de red
