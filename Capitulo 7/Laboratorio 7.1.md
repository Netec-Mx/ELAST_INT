**Objetivo:** Utilizar **ES\|QL** para realizar consultas tabulares,
generar reportes analíticos y construir visualizaciones en Kibana
mediante un enfoque basado en pipelines.

**7.1 Generalidades de ES\|QL — Consultas tabulares**

**En donde:**

ES\|QL es un lenguaje de consulta basado en **pipelines (\|)**, donde:

- cada comando procesa los datos

- la salida se convierte en entrada del siguiente paso

Conceptos clave:

- trabaja como tabla (filas/columnas)

- no usa JSON anidado como Query DSL

- optimizado para análisis (no escritura)

Sintaxis básical:

FROM logs  
\| WHERE @timestamp \> NOW() - 1 DAY  
\| STATS total = COUNT(\*) BY host.name

Esto refleja el flujo:

1.  fuente

2.  filtro

3.  agregación

------------------------------------------------------------------------

**Actividad 1 — Consulta tabular básica**

FROM operations-\*  
\| LIMIT 10

**En donde:**

- FROM: define el índice

- LIMIT: controla número de filas

Resultado:

- vista tipo tabla

- equivalente a SELECT \*

------------------------------------------------------------------------

**7.2 Sintaxis básica — Reportes ad-hoc**

**En donde:**

Estructura general:

FROM indice  
\| comando  
\| comando

Comandos principales:

- WHERE → filtrar

- KEEP → seleccionar columnas

- EVAL → crear campos

- SORT → ordenar

- LIMIT → limitar

------------------------------------------------------------------------

**Actividad 2 — Reporte rápido de transacciones**

FROM operations-\*  
\| KEEP custid, amount, date  
\| SORT amount DESC  
\| LIMIT 5

**En donde:**

- KEEP: reduce columnas → mejora rendimiento

- SORT: ordena resultados

- LIMIT: evita sobrecarga

Uso:

- reportes ad-hoc

- exploración rápida

------------------------------------------------------------------------

**Actividad 3 — Logs recientes**

FROM logs-\*  
\| WHERE @timestamp \> NOW() - 1 DAY  
\| KEEP @timestamp, host.name, message

**En donde:**

- NOW() - 1 DAY: filtro temporal

- ejecutado directamente en shards (optimización interna)

------------------------------------------------------------------------

**7.3 Operadores lógicos y condiciones**

**En donde:**

Operadores:

- AND, OR, NOT

- comparadores: ==, !=, \>, \<

- texto: LIKE

------------------------------------------------------------------------

**Actividad 4 — Filtrado de errores en logs**

FROM logs-\*  
\| WHERE status \>= 400 AND service.name == "auth-service"  
\| KEEP @timestamp, status, service.name

**En donde:**

- AND: ambas condiciones deben cumplirse

- filtro se aplica temprano → mejora rendimiento

------------------------------------------------------------------------

**Actividad 5 — Clasificación con EVAL**

FROM logs-\*  
\| WHERE status \>= 400  
\| EVAL tipo_error = CASE(  
status == 404, "No Encontrado",  
status \>= 500, "Error Servidor",  
"Otro"  
)  
\| WHERE NOT tipo_error == "Otro"

**En donde:**

- EVAL: crea campo dinámico

- CASE: lógica condicional

- WHERE NOT: elimina resultados

------------------------------------------------------------------------

**7.4 Agregaciones y funciones estadísticas**

**En donde:**

Comando principal:

- STATS

Funciones:

- COUNT, AVG, SUM, PERCENTILE

------------------------------------------------------------------------

**Actividad 6 — Agregación por cliente**

FROM operations-\*  
\| STATS total = COUNT(\*), promedio = AVG(amount) BY custid

**En donde:**

- agrupa por cliente

- calcula métricas

------------------------------------------------------------------------

**Actividad 7 — Análisis de logs por severidad**

FROM logs-\*  
\| WHERE status \>= 400  
\| EVAL severity = CASE(  
status \>= 500, "CRITICAL",  
status \>= 400, "ERROR",  
"INFO"  
)  
\| STATS total = COUNT(\*) BY severity

**En donde:**

- combinación de transformación + agregación

- patrón típico en observabilidad

------------------------------------------------------------------------

**Actividad 8 — Métricas avanzadas**

FROM logs-\*  
\| STATS  
total = COUNT(\*),  
avg_response = AVG(response_time_ms),  
p99 = PERCENTILE(response_time_ms, 99)  
BY service.name

**En donde:**

- percentiles → análisis de latencia

- útil en monitoreo

------------------------------------------------------------------------

**7.5 Exportación de resultados y visualización**

**En donde:**

Proceso:

1.  ejecutar consulta en Kibana Discover

2.  visualizar resultados

3.  exportar a CSV

------------------------------------------------------------------------

**Actividad 9 — Preparar datos para exportación**

FROM operations-\*  
\| WHERE amount \> 1000  
\| KEEP custid, amount, date  
\| SORT amount DESC

**En donde:**

- datos ya procesados → exportación limpia

- evita exportar datos crudos

------------------------------------------------------------------------

**7.6 Mejores prácticas**

**En donde: técnica**

Principios clave:

1.  Filtrar primero (WHERE)

2.  Reducir columnas (KEEP)

3.  Evitar cálculos complejos innecesarios

4.  Usar índices específicos

------------------------------------------------------------------------

**Actividad 10 — Consulta optimizada**

FROM operations-2026.04.\*  
\| WHERE amount \> 1000  
\| KEEP custid, amount  
\| STATS total = SUM(amount) BY custid

**En donde:**

- reduce dataset temprano

- mejora uso de CPU/memoria

------------------------------------------------------------------------

**7.7 Introducción a gráficos con Kibana**

**En donde:**

ES\|QL se integra directamente con:

Kibana

Permite:

- usar ES\|QL en Discover

- convertir resultados en visualizaciones

- crear dashboards

(Integración explicada en página 22 )

------------------------------------------------------------------------

**Actividad 11 — Crear dataset para gráfica**

FROM logs-\*  
\| WHERE @timestamp \> NOW() - 1 DAY  
\| STATS total = COUNT(\*) BY service.name

**Pasos en Kibana**

1.  Ir a **Discover**

2.  Ejecutar ES\|QL

3.  Seleccionar “Visualize”

4.  Elegir tipo de gráfico (barra/pie)

------------------------------------------------------------------------

**Actividad 12 — Serie temporal**

FROM logs-\*  
\| STATS total = COUNT(\*) BY DATE_TRUNC(1 hour, @timestamp)  
\| SORT total DESC

**En donde:**

- DATE_TRUNC: agrupa por tiempo

- base para dashboards

------------------------------------------------------------------------

**Cierre técnico del módulo**

ES\|QL representa un cambio importante porque:

- elimina complejidad del Query DSL

- permite análisis tipo SQL

- ejecuta pipelines optimizados directamente en el motor

Internamente:

- parser → AST

- optimizador → pushdown de filtros

- ejecución distribuida en shards
