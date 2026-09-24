# Stack Integral de Observabilidad: Docker, Hardware, Logs y Uptime

Este repositorio contiene la arquitectura completa para desplegar un sistema de monitoreo de grado de producción. Proporciona visibilidad total sobre contenedores, consumo de hardware, métricas internas del motor de Docker, centralización de logs, monitoreo de disponibilidad web (uptime/SSL) y un sistema de alertas automatizadas.

## Arquitectura del Stack

El ecosistema se divide en dos entornos:
*   **Servidor Central (Monitoring Hub):** Aloja Prometheus (Base temporal), Grafana (Visualización), Loki (Base de logs), Blackbox Exporter (Sondeos web) y Alertmanager (Gestión de notificaciones).
*   **Servidor(es) Remoto(s) (Nodos a monitorear):** Ejecutan cAdvisor (Contenedores), Node Exporter (Hardware), Promtail (Envío de logs) y exponen la API nativa del motor de Docker.

---

## Estructura del Directorio

```text
.
├── alertmanager
│   └── config.yml
├── nodo_remoto
│   ├── docker-compose.yml
│   └── promtail-config.yaml
├── docker-compose.yml (Central)
├── loki-config.yaml
├── prometheus
│   ├── alerts.yml
│   └── prometheus.yml
└── .env
```

---

## 1. Configuración Previa en Nodo Remoto (Docker Daemon)

Para que Prometheus pueda leer la salud interna del motor de Docker en los servidores remotos, debes habilitar la exposición de métricas nativas.

1. Edita o crea el archivo `/etc/docker/daemon.json` en el servidor remoto:
   ```json
   {
     "metrics-addr": "0.0.0.0:9323"
   }
   ```
2. Reinicia el servicio de Docker para aplicar el cambio:
   ```bash
   sudo systemctl restart docker
   ```

---

## 2. Instalación en los Servidores Remotos (Agentes)

El stack remoto recopila toda la información de la máquina y la expone (o envía) al servidor central.

1. Transfiere la carpeta `cadvisor/` a tu servidor destino.
2. Abre el archivo `promtail-config.yaml` y asegúrate de cambiar la URL de Loki para que apunte a la IP de tu Servidor Central: `http://IP_CENTRAL:3100/loki/api/v1/push`.
3. Levanta los agentes (cAdvisor en el puerto `9191`, Node Exporter en `9100` y Promtail):
   ```bash
   docker compose up -d
   ```

*(Nota de cAdvisor: Utiliza el flag `-disable_metrics=disk,tcp,udp,referenced_memory` para evitar bugs de uso excesivo de CPU en versiones modernas de Docker).*

---

## 3. Instalación del Servidor Central (Core y Alertas)

### A. Configuración de Alertas por Correo (Alertmanager)
Edita el archivo `alertmanager/config.yml` para configurar tu servidor SMTP (Ej. Gmail con Contraseña de Aplicación) y los correos de destino:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'tu-correo@gmail.com'
  smtp_auth_username: 'tu-correo@gmail.com'
  smtp_auth_password: 'tu-password-de-aplicacion'
  smtp_require_tls: true

route:
  receiver: 'equipo_soporte'

receivers:
  - name: 'equipo_soporte'
    email_configs:
      - to: 'admin@empresa.com, alertas@empresa.com'
        send_resolved: true
```

### B. Reglas de Alerta (Prometheus)
El archivo `prometheus/alerts.yml` define los disparadores. Por defecto incluye:
*   `ServidorCaido`: Se activa si un Node Exporter deja de responder por 1 minuto.
*   `WebCaida`: Se activa si Blackbox Exporter detecta un código HTTP distinto a 200 OK.

### C. Configuración de Raspado (Prometheus)
Edita el archivo `prometheus/prometheus.yml` asegurándote de utilizar el formato de corchetes `[]` para los targets y así evitar problemas de indentación YAML. Reemplaza `192.168.1.17` por la IP de tu servidor remoto, y agrega tus webs a Blackbox:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - '/etc/prometheus/alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['192.168.1.17:9191']
        
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.1.17:9100']

  - job_name: 'docker'
    static_configs:
      - targets: ['192.168.1.17:9323']

  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets: ["[https://mi-sitio-web.com](https://mi-sitio-web.com)", "[https://api.empresa.com](https://api.empresa.com)"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 'blackbox-exporter:9115'
```

### D. Levantar el Core Central
Configura tu archivo `.env` copiando el ejemplo (`cp env.example .env`) y establece tu clave `GF_SECURITY_ADMIN_PASSWORD`. Luego, levanta todo el ecosistema:

```bash
docker compose up -d
```

---

## 4. Dashboards de Grafana (Plantillas Nativas)

Ingresa a Grafana (`http://TU_IP_CENTRAL:3000`). En el apartado de Data Sources, añade **Prometheus** (`http://prometheus:9090`) y **Loki** (`http://loki:3100`).

Para visualizar la infraestructura sin diseñar paneles desde cero, ve a *Dashboards > Import* y utiliza los IDs correspondientes a estas plantillas:

*   **Node Exporter Full** (ID: 1860): Métricas completas de hardware del servidor físico o máquina virtual (Consumo de CPU, RAM, Disco y Red).
*   **Blackbox Exporter (HTTP prober)** (ID: 11529): Monitoreo sintético externo. Mide el Uptime de páginas web, tiempos de resolución DNS y alerta sobre la caducidad de certificados SSL/TLS.
*   **Cadvisor exporter** (ID: 14282): Métricas base crudas de uso de recursos extraídas directamente por el agente cAdvisor.
*   **Docker-cAdvisor** (ID: 11600 o 193): Vista combinada e intuitiva del rendimiento general de todos los contenedores Docker en ejecución.

---

## 5. Consultas PromQL Útiles (Solución de problemas)

Si deseas crear paneles personalizados o nuevas alertas de caída de contenedores, puedes usar estas sentencias en PromQL:

**Detección de Reinicios en contenedores (Últimos 15 minutos):**
```promql
changes(container_start_time_seconds{name!=""}[15m])
```
*(Si el valor es > 0, el contenedor se reinició).*

**Tiempo de Actividad (Uptime) de un contenedor:**
```promql
time() - container_start_time_seconds{name!=""}
```
*(Útil para crear alertas de "No Data" en Grafana si el contenedor colapsa y deja de enviar información).*
