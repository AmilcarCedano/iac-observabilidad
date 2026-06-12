# Instrucciones para validar el laboratorio

**Alumno:** Amilcar Cedano
**Curso:** Infraestructura como Código
**Repositorio:** https://github.com/AmilcarCedano/iac-observabilidad

## 1. Requisitos

- Docker Desktop funcionando (`docker --version`, `docker compose version`).
- Puertos libres: 3000, 3001, 3100, 8080, 8081, 9090, 9100, 12345.

## 2. Levantar el stack

```bash
git clone https://github.com/AmilcarCedano/iac-observabilidad.git
cd iac-observabilidad
docker compose up -d --build
docker compose ps   # los 8 contenedores deben estar "Up"
```

> **Nota (Windows):** este repositorio incluye un fix para Docker Desktop en Windows:
> el montaje de node-exporter se cambió de `/:/host:ro,rslave` a `/:/host:ro`, porque la
> propagación `rslave` no está soportada en Docker Desktop/WSL2 y el contenedor no arrancaba.

## 3. Verificar servicios

| Servicio | URL | Qué se debe ver |
|---|---|---|
| Frontend | http://localhost:8080 | "Hello World" con botones de tráfico/carga |
| Backend (métricas) | http://localhost:3001/metrics | métricas en formato Prometheus |
| Grafana | http://localhost:3000 | login `admin` / `admin` |
| Prometheus | http://localhost:9090/targets | 5 targets en estado UP |
| Loki | http://localhost:3100/ready | `ready` |
| Alloy | http://localhost:12345 | componentes Healthy |

## 4. Ver el dashboard

1. Entrar a Grafana (`admin`/`admin`) → **Dashboards** → **"Observabilidad — Amilcar Cedano"**.
2. Contiene 4 paneles:
   - **CPU contenedor backend (%)** — con umbral rojo en 50 (líneas).
   - **CPU del host (%)** — métrica de infraestructura vía node-exporter.
   - **Logs de aplicación (API + frontend)** — Loki, `{tier="application"} | json | level="ERROR"` (filtro por nivel).
   - **Logs de infraestructura** — Loki, `{tier="infrastructure"}`.

> **Nota sobre la métrica de CPU del contenedor:** la consulta original de la guía
> (`container_cpu_usage_seconds_total{name="lab-backend"}`) devuelve vacío en Docker Desktop
> para Windows porque cAdvisor no puede resolver los nombres de los contenedores en esa
> plataforma (limitación conocida). Como alternativa equivalente se usa la métrica que el
> propio backend expone en `/metrics`:
> `rate(backend_process_cpu_seconds_total[1m]) * 100` (% de CPU del proceso, 100 ≈ 1 núcleo).
> En Linux nativo la consulta original funciona sin cambios.

## 5. Probar la alarma (CPU > 50%)

1. La regla **"CPU backend > 50%"** está en **Alerting → Alert rules** (carpeta `laboratorio`,
   grupo `grupo-cpu`, evaluación cada 10s, pending period 30s, label `severity=warning`).
2. Generar carga (dos veces para superar el umbral con claridad):
   ```bash
   curl "http://localhost:3001/load?seconds=120"
   curl "http://localhost:3001/load?seconds=120"
   ```
3. Observar: el panel de CPU del backend supera el 50% y la regla pasa
   **Normal → Pending → Firing** en ~40–60 segundos.
4. Al terminar la carga, la métrica baja y la alarma vuelve a **Normal**.

## 6. Verificar el ciclo alarma → log (webhook)

1. El contact point **`backend-webhook`** (Alerting → Contact points) apunta a
   `http://backend:3001/alerts` y está asignado a la regla.
2. Cuando la alarma entra en Firing, el backend registra el log `grafana_alert_received`.
3. Verlo en el dashboard con la consulta Loki:
   ```logql
   {tier="application"} | json | msg=`grafana_alert_received`
   ```
   Aparece con `alert_status: firing` y `alertname: CPU backend > 50%`.

## 7. Reset

```bash
docker compose down      # detiene y conserva dashboards/alarmas
docker compose down -v   # borra también datos, dashboards y alarmas
```
