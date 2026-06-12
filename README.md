# Laboratorio de Observabilidad — Grafana, Prometheus y Loki

**Curso:** Infraestructura como Código
**Alumno:** Amilcar Cedano
**Base:** laboratorio del curso ([UPAO-Recursos/iac-observabilidad](https://github.com/UPAO-Recursos/iac-observabilidad))

Stack completo de observabilidad definido como código: métricas con **Prometheus**
(node-exporter + cAdvisor + `/metrics` de las apps), logs con **Loki** recolectados por
**Grafana Alloy**, y visualización + alarmas con **Grafana**. Todo se levanta con un solo
comando porque la configuración vive en archivos versionados, no en clics.

## Lo que se hizo en este laboratorio

- ✅ Stack levantado y verificado (8 contenedores, 5 targets UP en Prometheus).
- ✅ **Fix para Windows**: node-exporter no arrancaba en Docker Desktop por la opción
  `rslave`; se corrigió en el `docker-compose.yml`.
- ✅ Dashboard **"Observabilidad — Amilcar Cedano"** con 4 paneles: CPU del backend
  (con umbral en 50%), CPU del host, logs de aplicación (con filtro por nivel) y logs
  de infraestructura. Exportado como código en [`grafana/dashboards/`](grafana/dashboards/).
- ✅ Alarma **CPU backend > 50%** (evaluación cada 10s, pending period 30s, `severity=warning`).
- ✅ Ciclo cerrado **alarma → webhook → log**: contact point hacia `http://backend:3001/alerts`;
  el log `grafana_alert_received` vuelve a verse en el dashboard.
- ✅ En Windows, la métrica de cAdvisor por contenedor no expone `name="lab-backend"`
  (limitación de Docker Desktop), así que el panel y la alarma usan la métrica del propio
  backend: `rate(backend_process_cpu_seconds_total[1m]) * 100`. Detalle en
  [VALIDACION.md](VALIDACION.md).

## Documentación del trabajo

| Archivo | Contenido |
|---|---|
| [VALIDACION.md](VALIDACION.md) | Instrucciones paso a paso para validar el laboratorio |
| [RESPUESTAS.md](RESPUESTAS.md) | Respuestas a las preguntas del laboratorio |
| [EVIDENCIAS.md](EVIDENCIAS.md) | Registro de actividades con sus capturas |
| [capturas/](capturas/) | Evidencias en imagen de cada punto |

## Cómo levantarlo

```bash
docker compose up -d --build
docker compose ps   # los 8 contenedores deben estar "Up"
```

## Servicios y URLs

| Servicio       | URL                         | Notas                                  |
|----------------|-----------------------------|----------------------------------------|
| Frontend       | http://localhost:8080       | Hello World + botones de tráfico/carga |
| Backend (API)  | http://localhost:3001       | `/api/hello`, `/metrics`, `/load`      |
| Grafana        | http://localhost:3000       | admin / admin                          |
| Prometheus     | http://localhost:9090       | datasource ya provisionado             |
| Loki           | http://localhost:3100       | datasource ya provisionado             |
| Alloy (UI)     | http://localhost:12345      | estado del recolector de logs          |
| cAdvisor       | http://localhost:8081       | métricas por contenedor                |
| node-exporter  | http://localhost:9100/metrics | métricas del host                    |

## Probar la alarma rápido

```bash
curl "http://localhost:3001/load?seconds=120"   # genera carga de CPU
# en ~40-60s la regla pasa Normal -> Pending -> Firing (Alerting -> Alert rules)
```

## Reset

```bash
docker compose down      # detiene (conserva dashboards/alarmas)
docker compose down -v   # borra también datos, dashboards y alarmas
```

> Nota de versiones: el tag `prom/prometheus:latest` apunta aún a la rama 2.x (LTS),
> por eso se fija `v3.8.1`. Promtail está EOL (2026-03-02); el recolector de logs
> es Grafana Alloy.
