# Docker Advanced - Materi Lanjutan

## Daftar Isi
1. [Docker Security](#1-docker-security)
2. [Docker Networking Advanced](#2-docker-networking-advanced)
3. [Docker Volumes Management](#3-docker-volumes-management)
4. [Container Orchestration - Docker Swarm](#4-container-orchestration---docker-swarm)
5. [Monitoring & Observability](#5-monitoring--observability)
6. [Private Image Registry](#6-private-image-registry)
7. [Health Checks & Probes](#7-health-checks--probes)
8. [Multi-Architecture Builds](#8-multi-architecture-builds)
9. [Development Workflow](#9-development-workflow)
10. [Cost Optimization](#10-cost-optimization)

---

## 1. Docker Security

### 1.1 Vulnerability Scanning

#### Install Docker Scout
```bash
# Docker Scout sudah terintegrasi dengan Docker Desktop
# Untuk CLI, pastikan Docker versi terbaru

# Scan image untuk vulnerabilities
docker scout cves nginx:latest

# Scan dengan output detail
docker scout cves --format sarif nginx:latest > report.json

# Compare dengan base image
docker scout compare --to nginx:alpine nginx:latest
```

#### Install Trivy untuk Scanning
```bash
# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Scan image
trivy image nginx:latest

# Scan dengan severity filter
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan Dockerfile
trivy config Dockerfile

# Generate report
trivy image --format json --output result.json nginx:latest
```

### 1.2 Running as Non-Root User

#### Dockerfile dengan Non-Root User
```bash
cat > Dockerfile.secure << 'EOF'
FROM node:20-alpine

# Buat user dan group
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser

# Set working directory
WORKDIR /app

# Copy dengan ownership
COPY --chown=appuser:appgroup package*.json ./

# Install dependencies sebagai root (jika perlu)
RUN npm install --production

# Copy application code
COPY --chown=appuser:appgroup . .

# Switch ke non-root user
USER appuser

# Expose port non-privileged (> 1024)
EXPOSE 3000

CMD ["node", "app.js"]
EOF
```

#### Best Practices Security Dockerfile
```bash
cat > Dockerfile.hardened << 'EOF'
# Use specific version, tidak latest
FROM python:3.11.7-slim-bookworm

# Update dan install dependencies minimal
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -m -u 1000 -s /bin/bash appuser

WORKDIR /app

# Copy dan set ownership
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:5000/health')"

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF
```

### 1.3 Docker Secrets Management

#### Setup Docker Secrets (Swarm mode)
```bash
# Initialize swarm
docker swarm init

# Create secret dari file
echo "mySecurePassword123" | docker secret create db_password -

# Create secret dari file
echo "dbUser123" > db_user.txt
docker secret create db_user db_user.txt
rm db_user.txt

# List secrets
docker secret ls

# Inspect secret (tidak menampilkan value)
docker secret inspect db_password
```

#### Menggunakan Secrets dalam Service
```bash
# Create service dengan secrets
docker service create \
  --name webapp \
  --secret db_password \
  --secret db_user \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  --env DB_USER_FILE=/run/secrets/db_user \
  myapp:latest
```

#### Docker Compose dengan Secrets
```yaml
cat > docker-compose-secrets.yml << 'EOF'
version: '3.8'

services:
  webapp:
    image: myapp:latest
    secrets:
      - db_password
      - db_user
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
      DB_USER_FILE: /run/secrets/db_user

  db:
    image: postgres:15
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  db_user:
    file: ./secrets/db_user.txt
EOF
```

### 1.4 Security Best Practices Checklist

```bash
# 1. Scan images regularly
trivy image myapp:latest

# 2. Use minimal base images
# Prefer: alpine, distroless
# FROM gcr.io/distroless/python3

# 3. Update packages
apt-get update && apt-get upgrade -y

# 4. Remove unnecessary tools
apt-get remove --purge wget curl && apt-get autoremove

# 5. Set read-only filesystem
docker run --read-only --tmpfs /tmp myapp:latest

# 6. Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp:latest

# 7. Use security scanning in CI/CD
# Add to .gitlab-ci.yml or GitHub Actions

# 8. Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1
docker pull nginx:latest

# 9. Limit resources
docker run --memory="512m" --cpus="1.0" myapp:latest

# 10. Use AppArmor or SELinux
docker run --security-opt apparmor=docker-default myapp:latest
```

---

## 2. Docker Networking Advanced

### 2.1 Network Drivers

#### Bridge Network (Default)
```bash
# Create custom bridge network
docker network create \
  --driver bridge \
  --subnet=172.18.0.0/16 \
  --ip-range=172.18.5.0/24 \
  --gateway=172.18.5.254 \
  custom-bridge

# Run container dengan custom network
docker run -d \
  --name web1 \
  --network custom-bridge \
  --ip 172.18.5.10 \
  nginx

# Connect container ke network tambahan
docker network connect bridge web1
```

#### Host Network
```bash
# Container menggunakan host network (no isolation)
docker run -d \
  --name web-host \
  --network host \
  nginx

# Performance terbaik, tapi less secure
```

#### Overlay Network (untuk Swarm)
```bash
# Create overlay network
docker network create \
  --driver overlay \
  --attachable \
  --subnet=10.0.0.0/24 \
  my-overlay

# List networks
docker network ls

# Run service di overlay network
docker service create \
  --name webapp \
  --network my-overlay \
  --replicas 3 \
  myapp:latest
```

#### Macvlan Network
```bash
# Create macvlan network
# Container dapat IP dari physical network
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

# Run container dengan macvlan
docker run -d \
  --name container1 \
  --network macvlan-net \
  --ip=192.168.1.100 \
  nginx
```

### 2.2 DNS Configuration

#### Custom DNS dalam Container
```bash
# Set custom DNS
docker run -d \
  --name web \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  --dns-search example.com \
  nginx

# Set DNS di docker-compose.yml
cat > docker-compose-dns.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - example.com
      - internal.local
EOF
```

#### Service Discovery
```bash
# Dalam custom network, containers bisa saling resolve by name
docker network create app-network

docker run -d --name db --network app-network postgres
docker run -d --name web --network app-network nginx

# Dari web container, bisa ping db by name
docker exec web ping db
```

### 2.3 Network Troubleshooting

```bash
# Inspect network
docker network inspect bridge

# Check container network settings
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container-name

# Test connectivity
docker run --rm --network container:web1 nicolaka/netshoot ping google.com

# Network debugging tools
docker run -it --rm --network app-network nicolaka/netshoot

# Di dalam netshoot container:
# ifconfig, ping, traceroute, nslookup, curl, wget, tcpdump, etc.
```

### 2.4 Network Security

```bash
# Create isolated network
docker network create --internal internal-net

# Containers di internal-net tidak bisa akses internet
docker run -d --name db --network internal-net postgres

# Frontend di bridge, backend di internal
docker network create frontend
docker network create --internal backend

docker run -d --name db --network backend postgres
docker run -d --name app --network backend myapp:latest
docker run -d --name nginx --network frontend nginx

# Connect app ke both networks
docker network connect frontend app
```

---

## 3. Docker Volumes Management

### 3.1 Volume Types

#### Named Volumes
```bash
# Create volume
docker volume create my-data

# Create dengan driver options
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/external \
  --opt o=bind \
  external-data

# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-data

# Use volume
docker run -d \
  --name app \
  -v my-data:/app/data \
  myapp:latest
```

#### Bind Mounts
```bash
# Mount host directory
docker run -d \
  --name web \
  -v /home/user/html:/usr/share/nginx/html:ro \
  nginx

# Mount dengan full path
docker run -d \
  --name app \
  --mount type=bind,source=/home/user/data,target=/data \
  myapp:latest
```

#### Tmpfs Mounts (Memory)
```bash
# Create tmpfs mount (data di RAM, hilang saat container stop)
docker run -d \
  --name app \
  --tmpfs /tmp:rw,size=100m,mode=1777 \
  myapp:latest

# Atau dengan --mount
docker run -d \
  --name app \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=100m \
  myapp:latest
```

### 3.2 Volume Backup & Restore

#### Backup Volume
```bash
# Method 1: Backup ke tar file
docker run --rm \
  -v my-data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/my-data-backup-$(date +%Y%m%d).tar.gz /data

# Method 2: Rsync backup
docker run --rm \
  -v my-data:/source:ro \
  -v $(pwd)/backup:/backup \
  instrumentisto/rsync-ssh \
  rsync -av /source/ /backup/

# Backup semua volumes
for vol in $(docker volume ls -q); do
  docker run --rm \
    -v $vol:/data:ro \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/${vol}-$(date +%Y%m%d).tar.gz -C /data .
done
```

#### Restore Volume
```bash
# Create new volume
docker volume create my-data-restored

# Restore dari backup
docker run --rm \
  -v my-data-restored:/data \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/my-data-backup-20240101.tar.gz -C /

# Atau dengan pipe
cat my-data-backup.tar.gz | docker run --rm -i \
  -v my-data-restored:/data \
  ubuntu tar xz -C /data
```

### 3.3 Volume Drivers

#### NFS Volume Driver
```bash
# Install NFS client
sudo apt-get install -y nfs-common

# Create NFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/share \
  nfs-volume

# Use NFS volume
docker run -d \
  --name app \
  -v nfs-volume:/data \
  myapp:latest
```

#### CIFS/SMB Volume
```bash
# Create CIFS volume
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt o=username=user,password=pass,vers=3.0 \
  --opt device=//192.168.1.100/share \
  smb-volume
```

### 3.4 Volume Management Best Practices

```bash
# Remove unused volumes
docker volume prune

# Remove specific volume
docker volume rm my-data

# Check volume usage
docker system df -v

# Label volumes for better management
docker volume create \
  --label environment=production \
  --label application=webapp \
  prod-data

# Find volumes by label
docker volume ls --filter label=environment=production

# Backup script automation
cat > backup-volumes.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/backups/docker-volumes"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

for volume in $(docker volume ls -q); do
    echo "Backing up volume: $volume"
    docker run --rm \
        -v $volume:/data:ro \
        -v $BACKUP_DIR:/backup \
        alpine tar czf /backup/${volume}-${DATE}.tar.gz -C /data .
done

# Keep only last 7 days
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed!"
EOF

chmod +x backup-volumes.sh
```

---

## 4. Container Orchestration - Docker Swarm

### 4.1 Swarm Setup

#### Initialize Swarm
```bash
# Initialize swarm manager
docker swarm init --advertise-addr 192.168.1.100

# Output akan memberikan token untuk join worker
# Simpan token ini!

# Get join token untuk worker
docker swarm join-token worker

# Get join token untuk manager
docker swarm join-token manager

# Join sebagai worker (di node lain)
docker swarm join --token SWMTKN-1-xxx 192.168.1.100:2377

# Leave swarm
docker swarm leave

# Force remove node (di manager)
docker node rm node-name --force
```

#### Manage Nodes
```bash
# List nodes
docker node ls

# Inspect node
docker node inspect node-name

# Update node availability
docker node update --availability drain node-name

# Label nodes untuk deployment targeting
docker node update --label-add type=compute node-name
docker node update --label-add environment=production node-name

# Promote worker to manager
docker node promote node-name

# Demote manager to worker
docker node demote node-name
```

### 4.2 Deploy Services

#### Create Service
```bash
# Simple service
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx

# Service dengan constraints
docker service create \
  --name webapp \
  --replicas 5 \
  --constraint 'node.labels.type==compute' \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  --reserve-cpu 0.25 \
  --reserve-memory 256M \
  myapp:latest

# Service dengan update config
docker service create \
  --name api \
  --replicas 3 \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-failure-action rollback \
  myapi:latest
```

#### Service Management
```bash
# List services
docker service ls

# Inspect service
docker service inspect web

# View service logs
docker service logs web

# Scale service
docker service scale web=5

# Update service
docker service update --image nginx:alpine web

# Rollback service
docker service rollback web

# Remove service
docker service rm web
```

### 4.3 Stack Deployment

#### Docker Stack YML
```yaml
cat > stack.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        max_attempts: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.type == web

  api:
    image: myapi:latest
    networks:
      - frontend
      - backend
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.type == database
      restart_policy:
        condition: on-failure

  visualizer:
    image: dockersamples/visualizer
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true

volumes:
  db-data:

secrets:
  db_password:
    external: true
EOF
```

#### Stack Commands
```bash
# Deploy stack
docker stack deploy -c stack.yml myapp

# List stacks
docker stack ls

# List stack services
docker stack services myapp

# List stack tasks
docker stack ps myapp

# Remove stack
docker stack rm myapp
```

### 4.4 Swarm Networking

```bash
# Create overlay network
docker network create \
  --driver overlay \
  --attachable \
  --subnet 10.0.0.0/24 \
  swarm-network

# Ingress network (default)
# Automatic load balancing untuk published ports

# Encrypted overlay network
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-network
```

### 4.5 Swarm Secrets & Configs

#### Secrets
```bash
# Create secret
echo "mypassword" | docker secret create db_pass -

# Use in service
docker service create \
  --name db \
  --secret db_pass \
  postgres

# Update secret (harus recreate)
echo "newpassword" | docker secret create db_pass_v2 -
docker service update \
  --secret-rm db_pass \
  --secret-add db_pass_v2 \
  db
```

#### Configs
```bash
# Create config
docker config create nginx_config nginx.conf

# Use in service
docker service create \
  --name web \
  --config source=nginx_config,target=/etc/nginx/nginx.conf \
  nginx

# Update config
docker config create nginx_config_v2 nginx-new.conf
docker service update \
  --config-rm nginx_config \
  --config-add source=nginx_config_v2,target=/etc/nginx/nginx.conf \
  web
```

---

## 5. Monitoring & Observability

### 5.1 Prometheus Setup

#### Docker Compose untuk Monitoring Stack
```yaml
cat > monitoring-stack.yml << 'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    networks:
      - monitoring
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring
    restart: unless-stopped

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:
EOF
```

#### Prometheus Configuration
```yaml
mkdir -p prometheus
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'docker-cluster'
    environment: 'production'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Load rules
rule_files:
  - '/etc/prometheus/rules/*.yml'

# Scrape configurations
scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # cAdvisor
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # Docker daemon (jika metrics enabled)
  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']
EOF
```

#### Prometheus Alert Rules
```yaml
mkdir -p prometheus/rules
cat > prometheus/rules/alerts.yml << 'EOF'
groups:
  - name: container_alerts
    interval: 30s
    rules:
      - alert: ContainerDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.instance }} is down"
          description: "Container has been down for more than 1 minute."

      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.name }}"
          description: "Container memory usage is above 90%"

      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.name }}"
          description: "Container CPU usage is above 80%"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Less than 10% disk space remaining"
EOF
```

### 5.2 Grafana Dashboards

#### Grafana Datasource
```yaml
mkdir -p grafana/datasources
cat > grafana/datasources/prometheus.yml << 'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
EOF
```

#### Dashboard Provisioning
```yaml
mkdir -p grafana/dashboards
cat > grafana/dashboards/dashboard.yml << 'EOF'
apiVersion: 1

providers:
  - name: 'Docker Monitoring'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
EOF
```

### 5.3 Loki untuk Log Aggregation

```yaml
cat > loki-stack.yml << 'EOF'
version: '3.8'

services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - loki

networks:
  monitoring:
    external: true

volumes:
  loki-data:
EOF
```

#### Loki Configuration
```yaml
mkdir -p loki
cat > loki/loki-config.yml << 'EOF'
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2020-05-15
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h

storage_config:
  boltdb:
    directory: /loki/index

  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
EOF
```

#### Promtail Configuration
```yaml
mkdir -p promtail
cat > promtail/promtail-config.yml << 'EOF'
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'logstream'
      - source_labels: ['__meta_docker_container_label_logging_jobname']
        target_label: 'job'
EOF
```

### 5.4 Application Instrumentation

#### Python Flask dengan Prometheus
```python
cat > app-monitored.py << 'EOF'
from flask import Flask, Response
from prometheus_client import Counter, Histogram, Gauge, generate_latest
import time
import random

app = Flask(__name__)

# Metrics
REQUEST_COUNT = Counter('app_requests_total', 'Total requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('app_request_latency_seconds', 'Request latency', ['endpoint'])
ACTIVE_REQUESTS = Gauge('app_active_requests', 'Active requests')

@app.before_request
def before_request():
    ACTIVE_REQUESTS.inc()
    
@app.after_request
def after_request(response):
    ACTIVE_REQUESTS.dec()
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.endpoint,
        status=response.status_code
    ).inc()
    return response

@app.route('/')
@REQUEST_LATENCY.labels(endpoint='/').time()
def hello():
    time.sleep(random.uniform(0.1, 0.5))
    return {'message': 'Hello World'}

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')

@app.route('/health')
def health():
    return {'status': 'healthy'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

---

## 6. Private Image Registry

### 6.1 Docker Registry Setup

#### Basic Registry
```bash
# Run registry container
docker run -d \
  -p 5000:5000 \
  --name registry \
  --restart=always \
  -v registry-data:/var/lib/registry \
  registry:2

# Test registry
curl http://localhost:5000/v2/_catalog
```

#### Registry dengan Authentication
```bash
mkdir -p registry/auth

# Create password file
docker run --rm \
  --entrypoint htpasswd \
  httpd:2 -Bbn admin password123 > registry/auth/htpasswd

# Run registry dengan auth
docker run -d \
  -p 5000:5000 \
  --name registry-secure \
  --restart=always \
  -v registry-data:/var/lib/registry \
  -v $(pwd)/registry/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2

# Login ke registry
docker login localhost:5000
```

#### Registry dengan TLS
```bash
mkdir -p registry/certs

# Generate self-signed certificate
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout registry/certs/domain.key \
  -x509 -days 365 \
  -out registry/certs/domain.crt \
  -subj "/CN=registry.local"

# Run registry dengan TLS
docker run -d \
  -p 443:5000 \
  --name registry-tls \
  --restart=always \
  -v registry-data:/var/lib/registry \
  -v $(pwd)/registry/certs:/certs \
  -v $(pwd)/registry/auth:/auth \
  -e "REGISTRY_HTTP_ADDR=0.0.0.0:5000" \
  -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt" \
  -e "REGISTRY_HTTP_TLS_KEY=/certs/domain.key" \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2
```

### 6.2 Harbor Registry

#### Harbor dengan Docker Compose
```yaml
# Download Harbor
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz
tar xzvf harbor-offline-installer-v2.10.0.tgz
cd harbor

# Edit harbor.yml
cat > harbor.yml << 'EOF'
hostname: registry.example.com

http:
  port: 80

https:
  port: 443
  certificate: /data/cert/server.crt
  private_key: /data/cert/server.key

harbor_admin_password: Harbor12345

database:
  password: root123
  max_idle_conns: 50
  max_open_conns: 1000

data_volume: /data

storage_service:
  ca_bundle:
  filesystem:
    maxthreads: 100

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor

_version: 2.10.0
EOF

# Install Harbor
sudo ./install.sh

# Akses Harbor di https://registry.example.com
# Default: admin / Harbor12345
```

### 6.3 Push & Pull Images

#### Tag dan Push ke Private Registry
```bash
# Tag image
docker tag myapp:latest localhost:5000/myapp:latest
docker tag myapp:latest localhost:5000/myapp:v1.0.0

# Push image
docker push localhost:5000/myapp:latest
docker push localhost:5000/myapp:v1.0.0

# List images di registry
curl -u admin:password123 http://localhost:5000/v2/_catalog
curl -u admin:password123 http://localhost:5000/v2/myapp/tags/list

# Pull dari private registry
docker pull localhost:5000/myapp:latest
```

#### Configure Insecure Registry
```bash
# Edit daemon.json
sudo nano /etc/docker/daemon.json

# Add:
{
  "insecure-registries": ["localhost:5000", "192.168.1.100:5000"]
}

# Restart Docker
sudo systemctl restart docker
```

### 6.4 Registry Cleanup

```bash
# Registry garbage collection
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml

# Delete image dari registry (via API)
curl -X DELETE -u admin:password123 \
  http://localhost:5000/v2/myapp/manifests/sha256:xxxxx

# Registry maintenance script
cat > registry-cleanup.sh << 'EOF'
#!/bin/bash

REGISTRY="localhost:5000"
USERNAME="admin"
PASSWORD="password123"

# Get all repositories
REPOS=$(curl -s -u $USERNAME:$PASSWORD http://$REGISTRY/v2/_catalog | jq -r '.repositories[]')

for repo in $REPOS; do
    echo "Processing repository: $repo"
    
    # Get all tags
    TAGS=$(curl -s -u $USERNAME:$PASSWORD http://$REGISTRY/v2/$repo/tags/list | jq -r '.tags[]')
    
    # Keep only last 5 tags, delete others
    TAG_COUNT=$(echo "$TAGS" | wc -l)
    if [ $TAG_COUNT -gt 5 ]; then
        DELETE_TAGS=$(echo "$TAGS" | head -n -5)
        for tag in $DELETE_TAGS; do
            echo "Deleting $repo:$tag"
            # Get manifest digest
            DIGEST=$(curl -s -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
                -u $USERNAME:$PASSWORD \
                http://$REGISTRY/v2/$repo/manifests/$tag | \
                grep Docker-Content-Digest | awk '{print $2}' | tr -d '\r')
            
            # Delete manifest
            curl -X DELETE -u $USERNAME:$PASSWORD \
                http://$REGISTRY/v2/$repo/manifests/$DIGEST
        done
    fi
done

# Run garbage collection
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
EOF

chmod +x registry-cleanup.sh
```

---

## 7. Health Checks & Probes

### 7.1 Dockerfile Health Checks

#### Basic Health Check
```dockerfile
FROM nginx:alpine

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
```

#### Advanced Health Check
```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .

# Custom health check script
COPY healthcheck.js /usr/local/bin/healthcheck.js
RUN chmod +x /usr/local/bin/healthcheck.js

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node /usr/local/bin/healthcheck.js

EXPOSE 3000
CMD ["node", "server.js"]
```

#### Health Check Script Examples

**Node.js Health Check**
```javascript
cat > healthcheck.js << 'EOF'
const http = require('http');

const options = {
  host: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', (err) => {
  console.error('Health check failed:', err);
  process.exit(1);
});

request.end();
EOF
```

**Python Health Check**
```python
cat > healthcheck.py << 'EOF'
#!/usr/bin/env python3
import sys
import urllib.request

try:
    response = urllib.request.urlopen('http://localhost:5000/health', timeout=2)
    if response.status == 200:
        sys.exit(0)
    else:
        sys.exit(1)
except Exception as e:
    print(f"Health check failed: {e}")
    sys.exit(1)
EOF

chmod +x healthcheck.py
```

### 7.2 Docker Compose Health Checks

```yaml
cat > docker-compose-health.yml << 'EOF'
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    ports:
      - "8000:8000"

  nginx:
    image: nginx:alpine
    depends_on:
      web:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/"]
      interval: 30s
      timeout: 5s
      retries: 3
    ports:
      - "80:80"
EOF
```

### 7.3 Health Check Monitoring

```bash
# Check container health status
docker ps --format "table {{.Names}}\t{{.Status}}"

# Inspect health check details
docker inspect --format='{{json .State.Health}}' container-name | jq

# Watch health status
watch -n 1 'docker ps --format "table {{.Names}}\t{{.Status}}"'

# Filter unhealthy containers
docker ps --filter health=unhealthy

# Health check logs
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' container-name
```

### 7.4 Custom Health Endpoints

#### Flask Health Endpoint
```python
cat > app-health.py << 'EOF'
from flask import Flask, jsonify
import psycopg2
import redis
import time

app = Flask(__name__)

def check_database():
    try:
        conn = psycopg2.connect(
            host="db",
            database="mydb",
            user="user",
            password="password",
            connect_timeout=3
        )
        conn.close()
        return True, "Database OK"
    except Exception as e:
        return False, f"Database Error: {str(e)}"

def check_redis():
    try:
        r = redis.Redis(host='redis', port=6379, socket_connect_timeout=3)
        r.ping()
        return True, "Redis OK"
    except Exception as e:
        return False, f"Redis Error: {str(e)}"

@app.route('/health')
def health():
    checks = {
        'database': check_database(),
        'redis': check_redis(),
        'timestamp': time.time()
    }
    
    all_healthy = all(check[0] for check in checks.values() if isinstance(check, tuple))
    
    response = {
        'status': 'healthy' if all_healthy else 'unhealthy',
        'checks': {
            'database': {
                'status': 'up' if checks['database'][0] else 'down',
                'message': checks['database'][1]
            },
            'redis': {
                'status': 'up' if checks['redis'][0] else 'down',
                'message': checks['redis'][1]
            }
        },
        'timestamp': checks['timestamp']
    }
    
    status_code = 200 if all_healthy else 503
    return jsonify(response), status_code

@app.route('/ready')
def ready():
    # Readiness check - is app ready to serve traffic?
    db_ok, _ = check_database()
    redis_ok, _ = check_redis()
    
    if db_ok and redis_ok:
        return jsonify({'status': 'ready'}), 200
    else:
        return jsonify({'status': 'not ready'}), 503

@app.route('/live')
def live():
    # Liveness check - is app alive?
    return jsonify({'status': 'alive'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

#### Express Health Endpoint
```javascript
cat > server-health.js << 'EOF'
const express = require('express');
const { Pool } = require('pg');
const redis = require('redis');

const app = express();

const pgPool = new Pool({
  host: 'db',
  database: 'mydb',
  user: 'user',
  password: 'password',
  max: 10,
  connectionTimeoutMillis: 3000
});

const redisClient = redis.createClient({
  host: 'redis',
  port: 6379
});

async function checkDatabase() {
  try {
    await pgPool.query('SELECT 1');
    return { status: 'up', message: 'Database OK' };
  } catch (error) {
    return { status: 'down', message: error.message };
  }
}

async function checkRedis() {
  return new Promise((resolve) => {
    redisClient.ping((err, reply) => {
      if (err) {
        resolve({ status: 'down', message: err.message });
      } else {
        resolve({ status: 'up', message: 'Redis OK' });
      }
    });
  });
}

app.get('/health', async (req, res) => {
  const dbHealth = await checkDatabase();
  const redisHealth = await checkRedis();
  
  const healthy = dbHealth.status === 'up' && redisHealth.status === 'up';
  
  const response = {
    status: healthy ? 'healthy' : 'unhealthy',
    checks: {
      database: dbHealth,
      redis: redisHealth
    },
    timestamp: Date.now()
  };
  
  res.status(healthy ? 200 : 503).json(response);
});

app.get('/ready', async (req, res) => {
  const dbHealth = await checkDatabase();
  const redisHealth = await checkRedis();
  
  if (dbHealth.status === 'up' && redisHealth.status === 'up') {
    res.status(200).json({ status: 'ready' });
  } else {
    res.status(503).json({ status: 'not ready' });
  }
});

app.get('/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF
```

---

## 8. Multi-Architecture Builds

### 8.1 Docker Buildx Setup

```bash
# Check buildx
docker buildx version

# Create builder instance
docker buildx create --name multiarch --driver docker-container --use

# Inspect builder
docker buildx inspect multiarch --bootstrap

# List available platforms
docker buildx inspect --bootstrap | grep Platforms
```

### 8.2 Multi-Platform Build

#### Simple Multi-Arch Build
```bash
# Build untuk multiple platforms
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t myuser/myapp:latest \
  --push \
  .

# Build dan load untuk platform lokal
docker buildx build \
  --platform linux/amd64 \
  -t myapp:latest \
  --load \
  .
```

#### Dockerfile untuk Multi-Arch
```dockerfile
cat > Dockerfile.multiarch << 'EOF'
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.21 AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM"

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build untuk target platform
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -o /app/server .

# Final stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

COPY --from=builder /app/server /usr/local/bin/server

EXPOSE 8080

CMD ["server"]
EOF
```

### 8.3 Multi-Stage Multi-Arch Build

```dockerfile
cat > Dockerfile.advanced << 'EOF'
# syntax=docker/dockerfile:1

ARG NODE_VERSION=20
ARG ALPINE_VERSION=3.19

# Stage 1: Build dependencies
FROM --platform=$BUILDPLATFORM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS deps

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Build application
FROM --platform=$BUILDPLATFORM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS builder

ARG TARGETARCH

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Production image
FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION}

ARG TARGETPLATFORM
RUN echo "Running on $TARGETPLATFORM"

RUN apk add --no-cache tini

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser

USER appuser

EXPOSE 3000

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
EOF
```

### 8.4 Build Script untuk CI/CD

```bash
cat > build-multiarch.sh << 'EOF'
#!/bin/bash

set -e

IMAGE_NAME="myuser/myapp"
VERSION=${1:-latest}

# Platforms to build
PLATFORMS="linux/amd64,linux/arm64,linux/arm/v7"

echo "Building $IMAGE_NAME:$VERSION for $PLATFORMS"

# Create and use buildx builder
docker buildx create --name multibuilder --use 2>/dev/null || docker buildx use multibuilder

# Build and push
docker buildx build \
  --platform $PLATFORMS \
  --tag $IMAGE_NAME:$VERSION \
  --tag $IMAGE_NAME:latest \
  --push \
  --progress=plain \
  --cache-from type=registry,ref=$IMAGE_NAME:buildcache \
  --cache-to type=registry,ref=$IMAGE_NAME:buildcache,mode=max \
  .

echo "Build complete!"

# Inspect manifest
docker buildx imagetools inspect $IMAGE_NAME:$VERSION
EOF

chmod +x build-multiarch.sh
```

### 8.5 GitHub Actions untuk Multi-Arch

```yaml
cat > .github/workflows/multiarch.yml << 'EOF'
name: Multi-Architecture Build

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKER_USERNAME }}/myapp
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64,linux/arm/v7
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/myapp:buildcache
          cache-to: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/myapp:buildcache,mode=max
EOF
```

### 8.6 Test Multi-Arch Images

```bash
# Pull image untuk specific platform
docker pull --platform linux/arm64 myuser/myapp:latest

# Run image untuk specific platform
docker run --platform linux/amd64 myuser/myapp:latest

# Inspect manifest
docker manifest inspect myuser/myapp:latest

# Create manifest manually (alternative method)
docker manifest create myuser/myapp:latest \
  myuser/myapp:amd64 \
  myuser/myapp:arm64 \
  myuser/myapp:armv7

docker manifest push myuser/myapp:latest
```

---

## 9. Development Workflow

### 9.1 Hot Reload Setup

#### Node.js dengan Nodemon
```dockerfile
cat > Dockerfile.dev << 'EOF'
FROM node:20-alpine

WORKDIR /app

# Install nodemon globally
RUN npm install -g nodemon

# Copy package files
COPY package*.json ./
RUN npm install

# Copy application
COPY . .

EXPOSE 3000

CMD ["nodemon", "--watch", ".", "--ext", "js,json", "server.js"]
EOF
```

```yaml
cat > docker-compose.dev.yml << 'EOF'
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: nodemon --watch . --ext js,json server.js
EOF
```

#### Python dengan Flask Auto-Reload
```dockerfile
cat > Dockerfile.dev.python << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install --no-cache-dir watchdog

COPY . .

EXPOSE 5000

ENV FLASK_ENV=development
ENV FLASK_DEBUG=1

CMD ["flask", "run", "--host=0.0.0.0", "--reload"]
EOF
```

#### React dengan Hot Module Replacement
```dockerfile
cat > Dockerfile.dev.react << 'EOF'
FROM node:20-alpine

WORKDIR /app

ENV PATH /app/node_modules/.bin:$PATH

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
EOF
```

```yaml
cat > docker-compose.react.yml << 'EOF'
version: '3.8'

services:
  react-app:
    build:
      context: .
      dockerfile: Dockerfile.dev.react
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - CHOKIDAR_USEPOLLING=true
      - WATCHPACK_POLLING=true
    stdin_open: true
    tty: true
EOF
```

### 9.2 Debugging Containers

#### Remote Debugging Node.js
```dockerfile
cat > Dockerfile.debug << 'EOF'
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000 9229

CMD ["node", "--inspect=0.0.0.0:9229", "server.js"]
EOF
```

```yaml
cat > docker-compose.debug.yml << 'EOF'
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.debug
    ports:
      - "3000:3000"
      - "9229:9229"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
EOF
```

#### VSCode Debug Configuration
```json
cat > .vscode/launch.json << 'EOF'
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker: Attach to Node",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "/app",
      "protocol": "inspector",
      "restart": true
    }
  ]
}
EOF
```

#### Python Remote Debugging
```python
cat > app-debug.py << 'EOF'
import debugpy

# Enable debugging
debugpy.listen(("0.0.0.0", 5678))
print("Waiting for debugger attach...")
debugpy.wait_for_client()

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    name = "World"
    return f'Hello {name}!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
```

### 9.3 Development Tools Container

```yaml
cat > docker-compose.tools.yml << 'EOF'
version: '3.8'

services:
  # Database GUI
  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db

  # Redis GUI
  redis-commander:
    image: rediscommander/redis-commander:latest
    ports:
      - "8081:8081"
    environment:
      - REDIS_HOSTS=local:redis:6379

  # Mailhog (Email testing)
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI

  # Portainer (Docker management)
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data

  # Nginx Proxy Manager
  nginx-proxy:
    image: 'jc21/nginx-proxy-manager:latest'
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - nginx-data:/data
      - nginx-letsencrypt:/etc/letsencrypt

volumes:
  portainer-data:
  nginx-data:
  nginx-letsencrypt:
EOF
```

### 9.4 Local Development Environment

```bash
cat > dev-setup.sh << 'EOF'
#!/bin/bash

echo "Setting up development environment..."

# Create network
docker network create dev-network 2>/dev/null || true

# Start databases
docker-compose -f docker-compose.dev.yml up -d db redis

# Wait for databases
echo "Waiting for databases to be ready..."
sleep 10

# Run migrations
docker-compose -f docker-compose.dev.yml run --rm app npm run migrate

# Start application
docker-compose -f docker-compose.dev.yml up -d app

# Start development tools
docker-compose -f docker-compose.tools.yml up -d

echo "Development environment is ready!"
echo "Application: http://localhost:3000"
echo "Adminer: http://localhost:8080"
echo "Redis Commander: http://localhost:8081"
echo "Mailhog: http://localhost:8025"
echo "Portainer: http://localhost:9000"
EOF

chmod +x dev-setup.sh
```

### 9.5 Testing in Containers

#### Unit Tests
```yaml
cat > docker-compose.test.yml << 'EOF'
version: '3.8'

services:
  test:
    build:
      context: .
      target: test
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=test
    command: npm test

  test-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_pass
    tmpfs:
      - /var/lib/postgresql/data
EOF
```

#### Integration Tests
```bash
cat > run-tests.sh << 'EOF'
#!/bin/bash

set -e

echo "Starting test environment..."
docker-compose -f docker-compose.test.yml up -d test-db

echo "Waiting for database..."
sleep 5

echo "Running tests..."
docker-compose -f docker-compose.test.yml run --rm test

echo "Cleaning up..."
docker-compose -f docker-compose.test.yml down -v

echo "Tests completed!"
EOF

chmod +x run-tests.sh
```

---

## 10. Cost Optimization

### 10.1 Resource Limits

#### Set Resource Constraints
```bash
# CPU limits
docker run -d \
  --name app \
  --cpus="1.5" \
  --cpu-shares=1024 \
  myapp:latest

# Memory limits
docker run -d \
  --name app \
  --memory="512m" \
  --memory-swap="1g" \
  --memory-reservation="256m" \
  myapp:latest

# Combined resource limits
docker run -d \
  --name app \
  --cpus="2" \
  --memory="1g" \
  --pids-limit=100 \
  myapp:latest
```

#### Docker Compose Resource Limits
```yaml
cat > docker-compose.resources.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
    restart: unless-stopped

  api:
    image: myapi:latest
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        max_attempts: 3

  db:
    image: postgres:15
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
EOF
```

### 10.2 Image Size Optimization

#### Multi-Stage Build for Smaller Images
```dockerfile
cat > Dockerfile.optimized << 'EOF'
# Build stage
FROM node:20 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build && \
    npm prune --production

# Production stage
FROM node:20-alpine

# Install only runtime dependencies
RUN apk add --no-cache tini

WORKDIR /app

# Copy only necessary files
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser && \
    chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
EOF
```

#### Distroless Images
```dockerfile
cat > Dockerfile.distroless << 'EOF'
# Build stage
FROM golang:1.21 AS builder

WORKDIR /app

COPY go.* ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Production stage with distroless
FROM gcr.io/distroless/static-debian11

COPY --from=builder /app/main /

EXPOSE 8080

USER nonroot:nonroot

ENTRYPOINT ["/main"]
EOF
```

#### Image Size Comparison Script
```bash
cat > compare-images.sh << 'EOF'
#!/bin/bash

echo "Building different image variants..."

# Standard image
docker build -f Dockerfile.standard -t app:standard .

# Alpine-based
docker build -f Dockerfile.alpine -t app:alpine .

# Distroless
docker build -f Dockerfile.distroless -t app:distroless .

# Multi-stage optimized
docker build -f Dockerfile.optimized -t app:optimized .

echo ""
echo "Image Size Comparison:"
echo "======================"
docker images | grep "app:" | awk '{printf "%-20s %10s\n", $1":"$2, $7" "$8}'
EOF

chmod +x compare-images.sh
```

### 10.3 Build Cache Optimization

#### Leverage Build Cache
```dockerfile
cat > Dockerfile.cache << 'EOF'
FROM node:20-alpine

WORKDIR /app

# Cache dependencies layer
COPY package*.json ./
RUN npm ci --only=production

# Copy source code (changes more frequently)
COPY . .

RUN npm run build

EXPOSE 3000

CMD ["node", "dist/index.js"]
EOF
```

#### BuildKit Cache Mounts
```dockerfile
cat > Dockerfile.buildkit << 'EOF'
# syntax=docker/dockerfile:1

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

# Use cache mount for npm cache
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

COPY . .

RUN --mount=type=cache,target=/root/.npm \
    npm run build

EXPOSE 3000

CMD ["node", "dist/index.js"]
EOF
```

#### Build with Registry Cache
```bash
# Build with cache from registry
docker buildx build \
  --cache-from type=registry,ref=myuser/myapp:buildcache \
  --cache-to type=registry,ref=myuser/myapp:buildcache,mode=max \
  -t myuser/myapp:latest \
  --push \
  .
```

### 10.4 Auto-Scaling with Swarm

```yaml
cat > stack-autoscale.yml << 'EOF'
version: '3.8'

services:
  web:
    image: myapp:latest
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.web.rule=Host(`example.com`)"
    networks:
      - app-network

  # Traefik for load balancing
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.swarmMode=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - app-network

networks:
  app-network:
    driver: overlay
EOF
```

#### Auto-Scale Script
```bash
cat > autoscale.sh << 'EOF'
#!/bin/bash

SERVICE_NAME="myapp_web"
MIN_REPLICAS=2
MAX_REPLICAS=10
CPU_THRESHOLD=70

while true; do
    # Get current replicas
    CURRENT=$(docker service ls --filter name=$SERVICE_NAME --format "{{.Replicas}}" | cut -d'/' -f1)
    
    # Get average CPU usage
    CPU_AVG=$(docker stats --no-stream --format "{{.CPUPerc}}" $(docker ps -q -f name=$SERVICE_NAME) | \
              sed 's/%//g' | awk '{sum+=$1} END {print sum/NR}')
    
    echo "Current replicas: $CURRENT, Avg CPU: $CPU_AVG%"
    
    # Scale up
    if (( $(echo "$CPU_AVG > $CPU_THRESHOLD" | bc -l) )) && [ $CURRENT -lt $MAX_REPLICAS ]; then
        NEW_REPLICAS=$((CURRENT + 1))
        echo "Scaling up to $NEW_REPLICAS replicas"
        docker service scale $SERVICE_NAME=$NEW_REPLICAS
    fi
    
    # Scale down
    if (( $(echo "$CPU_AVG < 30" | bc -l) )) && [ $CURRENT -gt $MIN_REPLICAS ]; then
        NEW_REPLICAS=$((CURRENT - 1))
        echo "Scaling down to $NEW_REPLICAS replicas"
        docker service scale $SERVICE_NAME=$NEW_REPLICAS
    fi
    
    sleep 60
done
EOF

chmod +x autoscale.sh
```

### 10.5 Storage Optimization

#### Volume Cleanup Automation
```bash
cat > cleanup-storage.sh << 'EOF'
#!/bin/bash

echo "Docker Storage Cleanup"
echo "======================"

# Show current usage
echo ""
echo "Current disk usage:"
docker system df

echo ""
echo "Cleaning up..."

# Remove stopped containers
echo "Removing stopped containers..."
docker container prune -f

# Remove dangling images
echo "Removing dangling images..."
docker image prune -f

# Remove unused volumes
echo "Removing unused volumes..."
docker volume prune -f

# Remove build cache
echo "Removing build cache..."
docker builder prune -f

# Remove unused networks
echo "Removing unused networks..."
docker network prune -f

echo ""
echo "After cleanup:"
docker system df

# Optional: More aggressive cleanup (use with caution)
read -p "Run aggressive cleanup? (removes ALL unused data) [y/N]: " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    docker system prune -a --volumes -f
    echo "Aggressive cleanup completed!"
fi
EOF

chmod +x cleanup-storage.sh
```

#### Scheduled Cleanup (Cron)
```bash
# Add to crontab
cat > /etc/cron.d/docker-cleanup << 'EOF'
# Clean docker resources every Sunday at 2 AM
0 2 * * 0 root /usr/local/bin/docker-cleanup.sh >> /var/log/docker-cleanup.log 2>&1
EOF

# Install cleanup script
sudo cp cleanup-storage.sh /usr/local/bin/docker-cleanup.sh
sudo chmod +x /usr/local/bin/docker-cleanup.sh
```

### 10.6 Network Optimization

#### Optimize Container Communication
```yaml
cat > docker-compose.optimized-network.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    networks:
      - frontend
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

  api:
    image: myapi:latest
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  cache:
    image: redis:alpine
    networks:
      - backend
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

  db:
    image: postgres:15-alpine
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: password
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

networks:
  frontend:
    driver: bridge
    driver_opts:
      com.docker.network.driver.mtu: 1500
  backend:
    driver: bridge
    internal: true
    driver_opts:
      com.docker.network.driver.mtu: 1500
EOF
```

### 10.7 Monitoring Resource Usage

```bash
cat > monitor-resources.sh << 'EOF'
#!/bin/bash

# Monitor script
echo "Container Resource Monitor"
echo "=========================="
echo ""

while true; do
    clear
    echo "Container Resource Usage - $(date)"
    echo "=================================="
    echo ""
    
    # CPU and Memory usage
    docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"
    
    echo ""
    echo "Disk Usage:"
    docker system df
    
    echo ""
    echo "Top 5 largest images:"
    docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | head -6
    
    sleep 5
done
EOF

chmod +x monitor-resources.sh
```

### 10.8 Cost Analysis Script

```bash
cat > cost-analysis.sh << 'EOF'
#!/bin/bash

echo "Docker Infrastructure Cost Analysis"
echo "===================================="
echo ""

# Count resources
CONTAINERS=$(docker ps -q | wc -l)
IMAGES=$(docker images -q | wc -l)
VOLUMES=$(docker volume ls -q | wc -l)
NETWORKS=$(docker network ls -q | wc -l)

echo "Resource Count:"
echo "  Containers: $CONTAINERS"
echo "  Images: $IMAGES"
echo "  Volumes: $VOLUMES"
echo "  Networks: $NETWORKS"
echo ""

# Storage usage
echo "Storage Analysis:"
docker system df -v
echo ""

# CPU and Memory allocation
echo "Resource Allocation:"
docker ps --format "table {{.Names}}\t{{.Status}}" | while read name status; do
    if [ "$name" != "NAMES" ]; then
        cpu=$(docker inspect $name --format='{{.HostConfig.NanoCpus}}' 2>/dev/null)
        mem=$(docker inspect $name --format='{{.HostConfig.Memory}}' 2>/dev/null)
        
        if [ "$cpu" != "0" ] || [ "$mem" != "0" ]; then
            echo "Container: $name"
            [ "$cpu" != "0" ] && echo "  CPU: $(echo "scale=2; $cpu/1000000000" | bc) cores"
            [ "$mem" != "0" ] && echo "  Memory: $(echo "scale=2; $mem/1024/1024" | bc) MB"
        fi
    fi
done
echo ""

# Optimization recommendations
echo "Optimization Recommendations:"
echo "=============================="

# Check for dangling images
DANGLING=$(docker images -f "dangling=true" -q | wc -l)
if [ $DANGLING -gt 0 ]; then
    echo "⚠ Found $DANGLING dangling images - run: docker image prune"
fi

# Check for stopped containers
STOPPED=$(docker ps -a -f "status=exited" -q | wc -l)
if [ $STOPPED -gt 0 ]; then
    echo "⚠ Found $STOPPED stopped containers - run: docker container prune"
fi

# Check for unused volumes
UNUSED_VOL=$(docker volume ls -qf dangling=true | wc -l)
if [ $UNUSED_VOL -gt 0 ]; then
    echo "⚠ Found $UNUSED_VOL unused volumes - run: docker volume prune"
fi

# Check large images
echo ""
echo "Largest Images (consider optimization):"
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -h -r | head -5

echo ""
echo "Run './cleanup-storage.sh' to free up space"
EOF

chmod +x cost-analysis.sh
```

### 10.9 Best Practices Summary

```bash
cat > best-practices.md << 'EOF'
# Docker Cost Optimization Best Practices

## 1. Image Optimization
- [ ] Use multi-stage builds
- [ ] Use alpine or distroless base images
- [ ] Minimize layers (combine RUN commands)
- [ ] Remove unnecessary files
- [ ] Use .dockerignore
- [ ] Compress application assets

## 2. Resource Management
- [ ] Set CPU limits appropriately
- [ ] Set memory limits to prevent OOM
- [ ] Use resource reservations for critical services
- [ ] Monitor resource usage regularly
- [ ] Implement auto-scaling for variable loads

## 3. Storage Optimization
- [ ] Regular cleanup of unused resources
- [ ] Use named volumes for persistence
- [ ] Implement volume backup strategy
- [ ] Compress logs and rotate them
- [ ] Use tmpfs for temporary data

## 4. Network Optimization
- [ ] Use internal networks for backend services
- [ ] Implement service mesh for complex architectures
- [ ] Use overlay networks efficiently in Swarm
- [ ] Optimize DNS resolution
- [ ] Enable HTTP/2 and compression

## 5. Build Optimization
- [ ] Use build cache effectively
- [ ] Implement registry cache
- [ ] Use BuildKit cache mounts
- [ ] Parallelize builds when possible
- [ ] Use dedicated build nodes

## 6. Runtime Optimization
- [ ] Use health checks
- [ ] Implement graceful shutdown
- [ ] Use read-only filesystems when possible
- [ ] Minimize the number of running containers
- [ ] Use container restart policies wisely

## 7. Monitoring & Alerting
- [ ] Set up resource usage monitoring
- [ ] Configure alerts for high usage
- [ ] Track container metrics over time
- [ ] Monitor cost trends
- [ ] Review and optimize regularly

## 8. Development Practices
- [ ] Use docker-compose for local development
- [ ] Implement CI/CD for automated builds
- [ ] Test resource limits before production
- [ ] Document resource requirements
- [ ] Regular security audits

## Cost Calculation Formula
```
Monthly Cost = (CPU cores × CPU cost) + (Memory GB × Memory cost) + 
               (Storage GB × Storage cost) + (Network GB × Network cost)
```

## Tools for Cost Management
- Docker system df - disk usage
- docker stats - real-time resource usage
- cAdvisor - container metrics
- Prometheus - monitoring and alerting
- Grafana - visualization

## Automation
- Schedule regular cleanup jobs
- Implement auto-scaling policies
- Automate image scanning and updates
- Set up cost alerts
- Use IaC for reproducibility
EOF
```

---

## Kesimpulan

### Ringkasan Materi Advanced

Anda telah mempelajari 10 topik advanced Docker:

1. ✅ **Docker Security** - Vulnerability scanning, non-root users, secrets management
2. ✅ **Docker Networking** - Advanced network drivers, DNS, troubleshooting
3. ✅ **Docker Volumes** - Backup/restore, volume drivers, management
4. ✅ **Docker Swarm** - Orchestration, stack deployment, auto-scaling
5. ✅ **Monitoring & Observability** - Prometheus, Grafana, Loki logging
6. ✅ **Private Registry** - Setup registry, Harbor, image management
7. ✅ **Health Checks** - Liveness/readiness probes, custom endpoints
8. ✅ **Multi-Architecture** - Buildx, cross-platform builds, CI/CD
9. ✅ **Development Workflow** - Hot reload, debugging, testing
10. ✅ **Cost Optimization** - Resource limits, image optimization, auto-scaling

### Checklist Production-Ready

```bash
cat > production-checklist.md << 'EOF'
# Production-Ready Docker Checklist

## Security
- [ ] Images scanned for vulnerabilities
- [ ] Running as non-root user
- [ ] Secrets properly managed
- [ ] TLS/SSL configured
- [ ] Network segmentation implemented
- [ ] Regular security updates scheduled

## Performance
- [ ] Resource limits configured
- [ ] Health checks implemented
- [ ] Caching strategy optimized
- [ ] Images optimized for size
- [ ] Network optimized
- [ ] Storage performance tested

## Reliability
- [ ] High availability configured
- [ ] Backup strategy implemented
- [ ] Disaster recovery plan documented
- [ ] Monitoring and alerting set up
- [ ] Log aggregation configured
- [ ] Auto-scaling policies defined

## Maintainability
- [ ] Documentation complete
- [ ] CI/CD pipeline automated
- [ ] Infrastructure as Code
- [ ] Version control for configs
- [ ] Rollback strategy defined
- [ ] Update process documented

## Compliance
- [ ] Data retention policies
- [ ] Audit logging enabled
- [ ] Access control configured
- [ ] Encryption at rest and in transit
- [ ] Compliance requirements met
- [ ] Regular compliance audits
EOF
```

### Useful Commands Reference

```bash
cat > quick-reference.sh << 'EOF'
#!/bin/bash

echo "Docker Advanced - Quick Reference"
echo "=================================="
echo ""
echo "Security:"
echo "  trivy image IMAGE"
echo "  docker scan IMAGE"
echo "  docker secret create NAME FILE"
echo ""
echo "Networking:"
echo "  docker network create --driver overlay NAME"
echo "  docker network inspect NAME"
echo ""
echo "Volumes:"
echo "  docker volume backup/restore"
echo "  docker volume create --driver local"
echo ""
echo "Swarm:"
echo "  docker stack deploy -c FILE STACK"
echo "  docker service scale SERVICE=N"
echo ""
echo "Monitoring:"
echo "  docker stats"
echo "  docker system df"
echo ""
echo "Multi-arch:"
echo "  docker buildx build --platform linux/amd64,linux/arm64"
echo ""
echo "Optimization:"
echo "  docker system prune -a"
echo "  docker image prune -a"
EOF

chmod +x quick-reference.sh
```

### Next Steps

1. **Kubernetes** - Container orchestration di scale besar
2. **Service Mesh** - Istio, Linkerd untuk microservices
3. **GitOps** - ArgoCD, Flux untuk deployment
4. **Observability** - OpenTelemetry, Jaeger untuk tracing
5. **Security** - Falco, OPA untuk policy enforcement

### Resources

- Docker Documentation: https://docs.docker.com
- Docker Hub: https://hub.docker.com
- Play with Docker: https://labs.play-with-docker.com
- Docker Community: https://forums.docker.com
- Awesome Docker: https://github.com/veggiemonk/awesome-docker

---

**Selamat! Anda telah menyelesaikan materi Docker Advanced!** 🎉

Dokumentasi ini dapat diupdate bagian per bagian sesuai kebutuhan.