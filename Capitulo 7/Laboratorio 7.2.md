# Práctica 13: Creando visualizaciones con Kibana

**Objetivo**

Construir visualizaciones en Kibana a partir de consultas ES\|QL,
permitiendo transformar datos tabulares en gráficos útiles para análisis
de desempeño, monitoreo y negocio.

## Duración aproximada:
- 45 minutos.
---------------------------------------------------------------------------

**Concepto clave**

Kibana permite convertir resultados de ES\|QL en visualizaciones
mediante:

1.  Consulta → ES\|QL

2.  Transformación → tabla

3.  Visualización → gráfico

4.  Integración → dashboard

Esto elimina la necesidad de usar directamente aggregations JSON.

------------------------------------------------------------------------

**Tipos de visualizaciones más comunes**

- Barras (comparación por categorías)

- Líneas (series temporales)

- Pie (distribución porcentual)

- Métricas (valores únicos)

- Tablas (detalle)

------------------------------------------------------------------------

**Actividad 1 — Visualización básica (tabla)**

**Objetivo**

Validar datos en formato tabular antes de graficar.

**Paso 1 — Ejecutar consulta**

FROM operations-\*  
\| KEEP custid, amount, date  
\| SORT amount DESC

**Paso 2 — En Kibana**

1.  Ir a **Discover**

2.  Cambiar a modo **ES\|QL**

3.  Ejecutar la consulta

**En donde:**

- Esta vista es la base de cualquier visualización

- Permite validar:

  - columnas

  - tipos de datos

  - volumen

------------------------------------------------------------------------

**Actividad 2 — Gráfico de barras (Top clientes)**

**Objetivo**

Comparar clientes por monto total.

**Consulta**

FROM operations-\*  
\| STATS total = SUM(amount) BY custid  
\| SORT total DESC

**Pasos en Kibana**

1.  Ejecutar consulta en Discover

2.  Seleccionar “Visualize”

3.  Elegir **Bar chart**

4.  Configurar:

    - Eje X → custid

    - Eje Y → total

**En donde:**

- STATS genera agregación

- cada fila → categoría

- ideal para ranking

------------------------------------------------------------------------

**Actividad 3 — Gráfico de pie (distribución de impacto)**

**Objetivo**

Visualizar proporción de transacciones.

**Consulta**

FROM operations-\*  
\| EVAL impacto = CASE(  
amount \< 1000, "bajo",  
amount \< 5000, "medio",  
"alto"  
)  
\| STATS total = COUNT(\*) BY impacto

**Pasos**

1.  Ejecutar consulta

2.  Visualize → Pie chart

3.  Configurar:

    - Segmento → impacto

    - Métrica → total

**En donde:**

- útil para distribución

- permite identificar concentración

------------------------------------------------------------------------

**Actividad 4 — Serie temporal (logs)**

**Objetivo**

Monitorear volumen de eventos en el tiempo.

**Consulta**

FROM logs-app-\*  
\| STATS total = COUNT(\*) BY DATE_TRUNC(1 minute, @timestamp)  
\| SORT total DESC

**Pasos**

1.  Ejecutar consulta

2.  Visualize → Line chart

3.  Configurar:

    - Eje X → tiempo

    - Eje Y → total

**En donde:**

- DATE_TRUNC agrupa por intervalo

- base para monitoreo en tiempo real

------------------------------------------------------------------------

**Actividad 5 — Latencia por servicio**

**Objetivo**

Analizar desempeño.

**Consulta**

FROM logs-app-\*  
\| STATS avg_latency = AVG(response_time_ms) BY service

**Visualización**

- Tipo: barras

- X: servicio

- Y: latencia promedio

**En donde:**

- identifica cuellos de botella

- útil en observabilidad

------------------------------------------------------------------------

**Actividad 6 — Dashboard combinado**

**Objetivo**

Integrar múltiples visualizaciones.

**Pasos**

1.  Crear varias visualizaciones:

    - errores por servicio

    - latencia promedio

    - volumen de transacciones

2.  Ir a **Dashboard**

3.  Crear nuevo dashboard

4.  Agregar visualizaciones

**En donde:**

- permite análisis integral

- cada gráfico responde a una pregunta

------------------------------------------------------------------------

**Actividad 7 — Filtros interactivos**

**Objetivo**

Permitir análisis dinámico.

**Pasos**

1.  En dashboard → Add filter

2.  Ejemplo:

    - service = payments

    - rango de fechas

**En donde:**

- filtra todas las visualizaciones

- no requiere modificar ES\|QL

------------------------------------------------------------------------

**Actividad 8 — Métricas clave (KPI)**

**Objetivo**

Mostrar indicadores principales.

**Consulta**

FROM operations-\*  
\| STATS total = SUM(amount)

**Visualización**

- tipo: Metric

**En donde:**

- muestra valor único

- ideal para dashboards ejecutivos

------------------------------------------------------------------------

**Actividad 9 — Tabla analítica**

**Objetivo**

Detalle por cliente.

**Consulta**

FROM operations-\*  
\| STATS total = SUM(amount), promedio = AVG(amount) BY custid  
\| SORT total DESC

**Visualización**

- tipo: Data Table

**En donde:**

- combina agregación + detalle

- útil para reportes

------------------------------------------------------------------------

**Buenas prácticas**

1.  Filtrar datos antes de visualizar

2.  Usar nombres claros en campos (EVAL)

3.  Evitar datasets grandes sin agregación

4.  Usar intervalos de tiempo adecuados

5.  Diseñar dashboards con propósito claro

------------------------------------------------------------------------

**Flujo recomendado**

1.  Crear consulta ES\|QL

2.  Validar en tabla

3.  Crear visualización

4.  Ajustar ejes

5.  Integrar en dashboard
