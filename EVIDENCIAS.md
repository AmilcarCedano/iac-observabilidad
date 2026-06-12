# Evidencias del laboratorio — Observabilidad con Grafana, Prometheus y Loki

**Alumno:** Amilcar Cedano
**Curso:** Infraestructura como Código
**Fecha:** 11/06/2026
**Repositorio:** https://github.com/AmilcarCedano/iac-observabilidad

> Registro de actividades realizadas. Cada punto indica qué se hizo y la captura que lo evidencia.
> Las capturas se guardan en la carpeta `capturas/` con el número indicado.

---

## Punto 1 — Levantamiento del stack ✔
Se levantó el stack completo de observabilidad desde la carpeta del proyecto con
`docker compose up -d --build`. Se construyeron las imágenes del backend y frontend y
arrancaron los 8 contenedores: lab-backend, lab-frontend, lab-grafana, lab-prometheus,
lab-loki, lab-alloy, lab-cadvisor y lab-node-exporter.

- **Captura 01** — `capturas/01-docker-compose-up.png`: terminal con los 8 contenedores "Started".

## Punto 2 — Verificación de servicios accesibles ✔
Se comprobó en el navegador que todos los servicios del stack responden correctamente.

- **Captura 02** — `capturas/02-frontend.png`: Frontend "Hello World" en http://localhost:8080 con los botones de tráfico y carga.
- **Captura 03** — `capturas/03-backend-metrics.png`: métricas del backend en formato Prometheus en http://localhost:3001/metrics.
- **Captura 04** — `capturas/04-prometheus-targets.png`: los 5 targets de Prometheus en estado UP (http://localhost:9090/targets).
- **Captura 05** — `capturas/05-alloy.png`: componentes de Grafana Alloy en estado Healthy (http://localhost:12345).

## Punto 3 — Fix de compatibilidad con Windows (commit al repositorio) ✔
node-exporter no arrancaba en Docker Desktop para Windows por la opción de propagación
`rslave` en el montaje del filesystem. Se corrigió en `docker-compose.yml` y se subió el
commit `fix: node-exporter compatible con Docker Desktop en Windows (sin rslave)`.

- **Captura 06** — `capturas/06-commit-fix.png`: terminal con el commit y push exitoso al repositorio.

## Punto 4 — Troubleshooting: métrica de CPU por contenedor ✔
La consulta de la guía `container_cpu_usage_seconds_total{name="lab-backend"}` devuelve
"Empty query result" porque cAdvisor no puede resolver los nombres de los contenedores en
Docker Desktop para Windows (limitación conocida de la plataforma). Como alternativa
equivalente se usa la métrica que el propio backend expone en `/metrics`:
`rate(backend_process_cpu_seconds_total[1m]) * 100`, que mide el % de CPU del proceso del
backend (100 ≈ un núcleo). Se verificó que devuelve datos correctamente.

- **Captura 08** — `capturas/08-plan-b-cpu.png`: consulta alternativa devolviendo la serie del backend (la consulta original devolvía "Empty query result").

## Punto 5 — Datasources provisionados como código ✔
Verificación en Grafana (Connections → Data sources) de que Prometheus y Loki ya existen
sin crearlos a mano, porque fueron provisionados desde `grafana/provisioning/datasources/`.
Test exitoso de ambos.

- **Captura 09** — `capturas/09-datasource-prometheus.png`: Prometheus con "Save & test" exitoso.
- **Captura 10** — `capturas/10-datasource-loki.png`: Loki con "Save & test" exitoso.

## Punto 6 — Panel: CPU del contenedor backend con umbral en 50% ✔
Panel Time series con la consulta de CPU del backend, unidad Percent (0-100) y umbral
rojo en 50 visible como línea.

- **Captura 11** — `capturas/11-panel-cpu-backend.png`: panel en edición con consulta, unidad y umbral 50.

## Punto 7 — Panel: CPU del host (infraestructura) ✔
Panel Time series con `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)`,
métrica de toda la máquina vía node-exporter.

- **Captura 12** — `capturas/12-panel-cpu-host.png`: panel del host con datos.

## Punto 8 — Panel: logs de aplicación con filtro por nivel ✔
Panel tipo Logs con fuente Loki, consulta `{tier="application"} | json` y filtro por nivel
`| level="ERROR"`.

- **Captura 13** — `capturas/13-panel-logs-app.png`: panel mostrando solo errores con la consulta visible.

## Punto 9 — Panel: logs de infraestructura ✔
Panel tipo Logs con fuente Loki y consulta `{tier="infrastructure"}` (logs de Prometheus,
Grafana, Loki, exporters).

- **Captura 14** — `capturas/14-panel-logs-infra.png`: panel con logs de los componentes del stack.

## Punto 10 — Dashboard guardado ✔
Dashboard "Observabilidad — Amilcar Cedano" con los 4 paneles.

- **Captura 15** — `capturas/15-dashboard-completo.png`: vista completa del dashboard con el nombre del alumno.

## Punto 11 — Alarma CPU > 50% ✔
Regla de alerta con la consulta de CPU del backend, Threshold IS ABOVE 50, evaluación cada
10s, pending period de 30s y etiqueta severity=warning.

- **Captura 16** — `capturas/16-alert-rule.png`: configuración de la regla.

## Punto 12 — Evidencia de la alarma en Firing ✔
Se generó carga con `curl "http://localhost:3001/load?seconds=120"`; el panel superó el 50%
y la regla pasó de Normal → Pending → Firing.

- **Captura 17** — `capturas/17-panel-sobre-50.png`: panel de CPU por encima del umbral.
- **Captura 18** — `capturas/18-alerta-firing.png`: regla en estado Firing.

## Punto 13 — Ciclo cerrado alarma → log vía webhook ✔
Contact point tipo Webhook hacia `http://backend:3001/alerts`. Al dispararse la alarma, el
backend registra el log `grafana_alert_received`, que vuelve a verse en el dashboard.

- **Captura 19** — `capturas/19-contact-point-webhook.png`: contact point configurado.
- **Captura 20** — `capturas/20-log-alerta-en-dashboard.png`: log `grafana_alert_received` en el panel de logs.

## Punto 14 — Repositorio con historial de commits ✔
El trabajo se subió en commits progresivos, uno por etapa (fix de compatibilidad, limpieza
del repo, dashboard como código, documentación de validación, respuestas y evidencias).

- **Captura 21** — repositorio en GitHub con el historial de commits (incluida en el documento PDF de entrega).
