# Respuestas a las preguntas del laboratorio

**Curso:** Infraestructura como Código
**Alumno:** Amilcar Cedano
**Laboratorio:** Observabilidad y monitoreo con Grafana, Prometheus y Loki

---

## 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?

Porque métricas y logs son señales distintas que responden preguntas distintas. Prometheus
almacena **métricas**: valores numéricos agregados en el tiempo (% de CPU, peticiones por
segundo, latencia). Sirven para saber **qué** está pasando y **cuánto**, pero no guardan el
detalle de cada evento. Los **logs** son líneas de texto con el contexto de cada evento
individual (qué pedido falló, con qué `trace_id`, qué excepción se lanzó). Prometheus no está
diseñado para almacenar ni consultar texto; Loki sí: indexa los logs por etiquetas (como
`tier=application`) y permite filtrarlos y parsearlos (`| json | level="ERROR"`). En resumen:
con Prometheus detectamos que la tasa de errores subió; con Loki averiguamos **por qué** subió.

## 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

La configuración queda **versionada, reproducible y auditable**. Al estar definida en archivos
dentro de `grafana/provisioning/`, cualquier persona que clone el repositorio y ejecute
`docker compose up -d --build` obtiene exactamente el mismo entorno, sin depender de que
alguien recuerde hacer clics en la interfaz. Si el entorno se destruye (`docker compose down -v`),
se reconstruye idéntico en segundos. Además, los cambios de configuración pasan por Git
(historial, revisión, rollback), que es justamente el principio de Infraestructura como Código:
la infraestructura se declara en archivos, no se configura a mano.

## 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

Porque miden cosas diferentes:

- **CPU contenedor** (`container_cpu_usage_seconds_total{name="lab-backend"}`, vía cAdvisor)
  mide solo el consumo del contenedor del backend. 100% equivale aproximadamente a un núcleo
  completo, así que el contenedor puede estar saturado mientras la máquina apenas se inmuta.
- **CPU host** (`node_cpu_seconds_total`, vía node-exporter) promedia el uso de **todos los
  núcleos** de la máquina y de **todos los procesos** (incluidos los demás contenedores, el
  sistema operativo, el navegador, etc.).

Por eso el backend puede marcar 90% (un núcleo quemado por `/load`) mientras el host marca,
por ejemplo, 15% (porque tiene 8 núcleos). Para alertar sobre una aplicación concreta usaría
la **métrica del contenedor**, porque aísla el comportamiento de esa aplicación; la del host
se contamina con todo lo demás que corre en la máquina y podría ocultar (o falsear) el problema.

## 4. ¿Qué diferencia hay entre el *evaluation interval* y el *pending period* de una alarma?

- El **evaluation interval** (10s en este laboratorio) es **cada cuánto tiempo** Grafana ejecuta
  la consulta y evalúa la condición de la regla (¿la CPU está por encima de 50%?).
- El **pending period** (30s) es **cuánto tiempo seguido** debe mantenerse verdadera la condición
  antes de que la alarma pase de estado *Pending* a *Firing*.

Es decir: el interval define la frecuencia de chequeo y el pending period define la persistencia
exigida. Con 10s/30s, la condición debe cumplirse en ~3 evaluaciones consecutivas antes de
disparar. Esto evita falsas alarmas por picos breves de CPU: un pico de 5 segundos cruzará el
umbral en una evaluación, pero la alarma nunca llegará a Firing porque no se sostiene 30 segundos.
