# Monitoring Nginx with Prometheus and Grafana: Complete Setup Guide

## Table of Contents
1. Prerequisites
2. System Architecture
3. Installation and Configuration
4. Dashboard Setup
5. Key Metrics and Alerts
6. Troubleshooting
7. Best Practices

## 1. Prerequisites

- Docker and Docker Compose installed
- Basic understanding of Nginx
- Root or sudo access
- Git (optional)

## 2. System Architecture

The monitoring stack consists of:
- Nginx with stub_status enabled
- Nginx Prometheus Exporter
- Prometheus
- Grafana

Data flow: Nginx → Nginx Exporter → Prometheus → Grafana

## 3. Installation and Configuration

### 3.1 Project Structure

Create a new directory and set up the following structure:
```bash
nginx-monitoring/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── default.conf
└── prometheus/
    └── prometheus.yml
```

### 3.2 Nginx Configuration

Create `nginx/nginx.conf`:
```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

Create `nginx/conf.d/default.conf`:
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /metrics {
        stub_status on;
        access_log off;
        allow all;
    }
}
```

### 3.3 Prometheus Configuration

Create `prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
    metrics_path: /metrics

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### 3.4 Docker Compose Configuration

Create `docker-compose.yml`:
```yaml
version: '3'

services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    networks:
      - monitoring

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    command:
      - -nginx.scrape-uri=http://nginx/metrics
    ports:
      - "9113:9113"
    depends_on:
      - nginx
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --web.console.libraries=/usr/share/prometheus/console_libraries
      - --web.console.templates=/usr/share/prometheus/consoles
    depends_on:
      - nginx-exporter
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SECURITY_ADMIN_USER=admin
    depends_on:
      - prometheus
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
```

### 3.5 Starting the Stack

1. Start all services:
```bash
docker-compose up -d
```

2. Verify services are running:
```bash
docker-compose ps
```

## 4. Dashboard Setup

1. Access Grafana at `http://localhost:3000`
2. Login with admin/admin
3. Add Prometheus data source:
   - Click Configuration → Data Sources
   - Add Prometheus
   - URL: `http://prometheus:9090`
   - Click "Save & Test"

4. Import Dashboard:
   - Click + → Import
   - Enter dashboard ID: 12708
   - Select Prometheus data source
   - Click Import

## 5. Key Metrics and Alerts

### 5.1 Essential Metrics
- `nginx_up`: Nginx availability
- `nginx_connections_active`: Current active connections
- `nginx_connections_reading`: Connections reading request body
- `nginx_connections_writing`: Connections writing response
- `nginx_connections_waiting`: Keep-alive connections
- `nginx_http_requests_total`: Total HTTP requests

### 5.2 Recommended Alerts

Create these alerts in Prometheus:

```yaml
groups:
- name: nginx
  rules:
  - alert: NginxDown
    expr: nginx_up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Nginx is down"
      
  - alert: HighConnectionCount
    expr: nginx_connections_active > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High number of active connections"
```

## 6. Troubleshooting

Common issues and solutions:

1. Nginx Exporter can't connect:
   - Check if Nginx `/metrics` endpoint is accessible
   - Verify network connectivity between containers
   - Check Nginx logs: `docker-compose logs nginx`

2. No metrics in Grafana:
   - Verify Prometheus targets are up
   - Check Prometheus logs: `docker-compose logs prometheus`
   - Verify data source configuration in Grafana

3. Dashboard not showing data:
   - Verify time range selection
   - Check Prometheus query in panel
   - Verify metrics exist in Prometheus

## 7. Best Practices

1. Security:
   - Change default Grafana password
   - Restrict `/metrics` endpoint access
   - Use HTTPS for production
   - Implement authentication

2. Performance:
   - Adjust scrape intervals based on needs
   - Monitor disk usage for Prometheus
   - Set appropriate retention periods

3. Maintenance:
   - Regular backups of Grafana dashboards
   - Monitor disk space for metrics storage
   - Keep components updated

4. Scaling:
   - Consider federation for large deployments
   - Use service discovery in production
   - Implement load balancing

This setup provides a solid foundation for monitoring Nginx. Adjust configurations and alerts based on your specific requirements.