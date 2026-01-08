# Tutorial Docker untuk Pemula - Ubuntu 24.04 Server/WSL

## Daftar Isi
1. [Instalasi Docker](#instalasi-docker)
2. [Konsep Dasar Docker](#konsep-dasar-docker)
3. [Docker untuk Nginx](#docker-untuk-nginx)
4. [Docker untuk MariaDB](#docker-untuk-mariadb)
5. [Docker untuk PHP](#docker-untuk-php)
6. [Docker untuk Python](#docker-untuk-python)
7. [Docker untuk Node.js](#docker-untuk-nodejs)
8. [Docker Compose](#docker-compose)
9. [Terraform untuk Docker](#terraform-untuk-docker)
10. [Ansible untuk Docker](#ansible-untuk-docker)
11. [CI/CD Pipeline](#cicd-pipeline)

---

## Instalasi Docker

### Langkah 1: Update Sistem
```bash
sudo apt update
sudo apt upgrade -y
```

### Langkah 2: Install Dependencies
```bash
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### Langkah 3: Tambahkan Docker Repository
```bash
# Tambahkan Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Setup repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Langkah 4: Install Docker Engine
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Langkah 5: Verifikasi Instalasi
```bash
sudo docker --version
sudo docker compose version
```

### Langkah 6: Jalankan Docker Tanpa Sudo (Opsional)
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Langkah 7: Test Docker
```bash
docker run hello-world
```

---

## Konsep Dasar Docker

### Apa itu Docker?
Docker adalah platform untuk mengembangkan, mengirim, dan menjalankan aplikasi dalam container. Container adalah unit standar software yang mengemas kode dan semua dependensinya.

### Istilah Penting:
- **Image**: Template read-only untuk membuat container
- **Container**: Instance yang berjalan dari sebuah image
- **Dockerfile**: File instruksi untuk membuat image
- **Docker Hub**: Registry publik untuk menyimpan image
- **Volume**: Penyimpanan persisten untuk data container
- **Network**: Komunikasi antar container

### Perintah Dasar Docker:
```bash
# Lihat image yang tersedia
docker images

# Lihat container yang berjalan
docker ps

# Lihat semua container (termasuk yang tidak berjalan)
docker ps -a

# Pull image dari Docker Hub
docker pull nama-image:tag

# Jalankan container
docker run nama-image

# Stop container
docker stop container-id

# Hapus container
docker rm container-id

# Hapus image
docker rmi image-id

# Lihat logs container
docker logs container-id

# Masuk ke dalam container
docker exec -it container-id /bin/bash
```

---

## Docker untuk Nginx

### Langkah 1: Buat Direktori Project
```bash
mkdir -p ~/docker-projects/nginx
cd ~/docker-projects/nginx
```

### Langkah 2: Buat File HTML Sederhana
```bash
mkdir -p html
cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Nginx</title>
</head>
<body>
    <h1>Selamat Datang di Docker Nginx!</h1>
    <p>Server ini berjalan di dalam Docker container.</p>
</body>
</html>
EOF
```

### Langkah 3: Jalankan Nginx Container
```bash
docker run -d \
  --name nginx-server \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:latest
```

Penjelasan:
- `-d`: Jalankan di background (detached mode)
- `--name`: Beri nama container
- `-p 8080:80`: Map port 8080 host ke port 80 container
- `-v`: Mount volume dari host ke container
- `nginx:latest`: Gunakan image nginx versi terbaru

### Langkah 4: Test Nginx
```bash
curl http://localhost:8080
```

### Langkah 5: Buat Custom Nginx Config (Opsional)
```bash
mkdir -p conf
cat > conf/default.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
}
EOF
```

Jalankan dengan custom config:
```bash
docker run -d \
  --name nginx-custom \
  -p 8081:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  -v $(pwd)/conf/default.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx:latest
```

---

## Docker untuk MariaDB

### Langkah 1: Buat Direktori Project
```bash
mkdir -p ~/docker-projects/mariadb
cd ~/docker-projects/mariadb
```

### Langkah 2: Jalankan MariaDB Container
```bash
docker run -d \
  --name mariadb-server \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_USER=myuser \
  -e MYSQL_PASSWORD=mypassword \
  -p 3306:3306 \
  -v mariadb-data:/var/lib/mysql \
  mariadb:latest
```

Penjelasan:
- `-e`: Set environment variables
- `MYSQL_ROOT_PASSWORD`: Password untuk user root
- `MYSQL_DATABASE`: Buat database otomatis
- `MYSQL_USER` & `MYSQL_PASSWORD`: Buat user non-root
- `-v mariadb-data:/var/lib/mysql`: Named volume untuk data persisten

### Langkah 3: Verifikasi MariaDB
```bash
# Lihat logs
docker logs mariadb-server

# Masuk ke MariaDB
docker exec -it mariadb-server mysql -u root -p
```

Di dalam MySQL prompt:
```sql
SHOW DATABASES;
USE mydb;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100));
INSERT INTO users (name) VALUES ('John Doe');
SELECT * FROM users;
EXIT;
```

### Langkah 4: Backup dan Restore

Backup:
```bash
docker exec mariadb-server mysqldump -u root -prootpassword mydb > backup.sql
```

Restore:
```bash
docker exec -i mariadb-server mysql -u root -prootpassword mydb < backup.sql
```

---

## Docker untuk PHP

### Langkah 1: Buat Direktori Project
```bash
mkdir -p ~/docker-projects/php
cd ~/docker-projects/php
```

### Langkah 2: Buat File PHP
```bash
mkdir -p www
cat > www/index.php << 'EOF'
<?php
phpinfo();
?>
EOF

cat > www/test.php << 'EOF'
<?php
echo "Hello from PHP in Docker!<br>";
echo "PHP Version: " . phpversion() . "<br>";
echo "Current Time: " . date('Y-m-d H:i:s') . "<br>";

// Test MySQL connection
$host = getenv('DB_HOST') ?: 'localhost';
$user = getenv('DB_USER') ?: 'root';
$pass = getenv('DB_PASS') ?: '';
$db = getenv('DB_NAME') ?: 'test';

try {
    $conn = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
    echo "Database connection: OK<br>";
} catch(PDOException $e) {
    echo "Database connection: FAILED - " . $e->getMessage() . "<br>";
}
?>
EOF
```

### Langkah 3: Buat Dockerfile untuk PHP
```bash
cat > Dockerfile << 'EOF'
FROM php:8.2-apache

# Install ekstensi PHP yang dibutuhkan
RUN docker-php-ext-install pdo pdo_mysql mysqli

# Enable Apache mod_rewrite
RUN a2enmod rewrite

# Copy aplikasi
COPY www/ /var/www/html/

# Set permission
RUN chown -R www-data:www-data /var/www/html

EXPOSE 80
EOF
```

### Langkah 4: Build dan Jalankan
```bash
# Build image
docker build -t my-php-app .

# Jalankan container
docker run -d \
  --name php-server \
  -p 8082:80 \
  -v $(pwd)/www:/var/www/html \
  my-php-app
```

### Langkah 5: Test PHP
```bash
curl http://localhost:8082/test.php
```

---

## Docker untuk Python

### Langkah 1: Buat Direktori Project
```bash
mkdir -p ~/docker-projects/python
cd ~/docker-projects/python
```

### Langkah 2: Buat Aplikasi Python (Flask)
```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        'message': 'Hello from Python Flask in Docker!',
        'version': '1.0.0',
        'environment': os.getenv('ENVIRONMENT', 'development')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
```

### Langkah 3: Buat Requirements
```bash
cat > requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
EOF
```

### Langkah 4: Buat Dockerfile
```bash
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# Copy requirements dan install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy aplikasi
COPY . .

EXPOSE 5000

# Jalankan dengan gunicorn untuk production
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
EOF
```

### Langkah 5: Build dan Jalankan
```bash
# Build image
docker build -t my-python-app .

# Jalankan container
docker run -d \
  --name python-server \
  -p 5000:5000 \
  -e ENVIRONMENT=production \
  my-python-app
```

### Langkah 6: Test Python App
```bash
curl http://localhost:5000/
curl http://localhost:5000/health
```

---

## Docker untuk Node.js

### Langkah 1: Buat Direktori Project
```bash
mkdir -p ~/docker-projects/nodejs
cd ~/docker-projects/nodejs
```

### Langkah 2: Buat Aplikasi Node.js (Express)
```bash
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/', (req, res) => {
    res.json({
        message: 'Hello from Node.js in Docker!',
        version: '1.0.0',
        timestamp: new Date().toISOString()
    });
});

app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
});
EOF
```

### Langkah 3: Buat package.json
```bash
cat > package.json << 'EOF'
{
  "name": "docker-nodejs-app",
  "version": "1.0.0",
  "description": "Node.js app in Docker",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
```

### Langkah 4: Buat Dockerfile
```bash
cat > Dockerfile << 'EOF'
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy aplikasi
COPY . .

EXPOSE 3000

CMD ["npm", "start"]
EOF
```

### Langkah 5: Buat .dockerignore
```bash
cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
.git
.gitignore
EOF
```

### Langkah 6: Build dan Jalankan
```bash
# Build image
docker build -t my-nodejs-app .

# Jalankan container
docker run -d \
  --name nodejs-server \
  -p 3000:3000 \
  my-nodejs-app
```

### Langkah 7: Test Node.js App
```bash
curl http://localhost:3000/
curl http://localhost:3000/health
```

---

## Docker Compose

Docker Compose memungkinkan kita mengelola multiple containers sekaligus.

### Langkah 1: Buat Direktori Project Lengkap
```bash
mkdir -p ~/docker-projects/full-stack
cd ~/docker-projects/full-stack
```

### Langkah 2: Struktur Direktori
```bash
mkdir -p nginx/conf nginx/html
mkdir -p php/www
mkdir -p python
mkdir -p nodejs
```

### Langkah 3: Buat docker-compose.yml
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Nginx Web Server
  nginx:
    image: nginx:latest
    container_name: nginx-webserver
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/html:/usr/share/nginx/html:ro
      - ./nginx/conf/default.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - app-network
    depends_on:
      - php
      - nodejs
      - python
    restart: unless-stopped

  # MariaDB Database
  mariadb:
    image: mariadb:latest
    container_name: mariadb-database
    environment:
      MYSQL_ROOT_PASSWORD: rootpass123
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass123
    ports:
      - "3306:3306"
    volumes:
      - mariadb-data:/var/lib/mysql
      - ./mariadb/init:/docker-entrypoint-initdb.d
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # PHP-FPM
  php:
    build: ./php
    container_name: php-backend
    volumes:
      - ./php/www:/var/www/html
    networks:
      - app-network
    depends_on:
      mariadb:
        condition: service_healthy
    environment:
      DB_HOST: mariadb
      DB_USER: appuser
      DB_PASS: apppass123
      DB_NAME: appdb
    restart: unless-stopped

  # Python Flask App
  python:
    build: ./python
    container_name: python-api
    ports:
      - "5000:5000"
    networks:
      - app-network
    environment:
      FLASK_ENV: production
      DB_HOST: mariadb
    restart: unless-stopped

  # Node.js Express App
  nodejs:
    build: ./nodejs
    container_name: nodejs-api
    ports:
      - "3000:3000"
    volumes:
      - ./nodejs:/app
      - /app/node_modules
    networks:
      - app-network
    environment:
      NODE_ENV: production
      DB_HOST: mariadb
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  mariadb-data:
    driver: local
EOF
```

### Langkah 4: Buat File Pendukung

#### Nginx Config
```bash
cat > nginx/conf/default.conf << 'EOF'
upstream php_backend {
    server php:9000;
}

upstream nodejs_backend {
    server nodejs:3000;
}

upstream python_backend {
    server python:5000;
}

server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html index.php;

    # Static files
    location / {
        try_files $uri $uri/ =404;
    }

    # PHP
    location ~ \.php$ {
        fastcgi_pass php_backend;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
        include fastcgi_params;
    }

    # Node.js API
    location /api/node/ {
        proxy_pass http://nodejs_backend/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Python API
    location /api/python/ {
        proxy_pass http://python_backend/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

#### PHP Dockerfile
```bash
cat > php/Dockerfile << 'EOF'
FROM php:8.2-fpm

RUN docker-php-ext-install pdo pdo_mysql mysqli

WORKDIR /var/www/html
EOF
```

#### Python Dockerfile
```bash
cat > python/Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF

cat > python/requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
pymysql==1.1.0
EOF

cat > python/app.py << 'EOF'
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return {'message': 'Python API is running'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

#### Node.js Dockerfile
```bash
cat > nodejs/Dockerfile << 'EOF'
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
EOF

cat > nodejs/package.json << 'EOF'
{
  "name": "nodejs-api",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.0"
  }
}
EOF

cat > nodejs/app.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.json({ message: 'Node.js API is running' });
});

app.listen(3000, '0.0.0.0', () => {
    console.log('Server running on port 3000');
});
EOF
```

### Langkah 5: Perintah Docker Compose

```bash
# Build semua services
docker compose build

# Jalankan semua services
docker compose up -d

# Lihat status services
docker compose ps

# Lihat logs
docker compose logs -f

# Stop semua services
docker compose down

# Stop dan hapus volumes
docker compose down -v

# Restart service tertentu
docker compose restart nginx

# Scale service
docker compose up -d --scale nodejs=3
```

---

## Terraform untuk Docker

Terraform memungkinkan Infrastructure as Code (IaC) untuk mengelola Docker containers.

### Langkah 1: Install Terraform
```bash
# Download Terraform
wget https://releases.hashicorp.com/terraform/1.6.6/terraform_1.6.6_linux_amd64.zip

# Extract
unzip terraform_1.6.6_linux_amd64.zip

# Move ke PATH
sudo mv terraform /usr/local/bin/

# Verifikasi
terraform --version
```

### Langkah 2: Buat Direktori Terraform
```bash
mkdir -p ~/docker-projects/terraform
cd ~/docker-projects/terraform
```

### Langkah 3: Buat main.tf
```bash
cat > main.tf << 'EOF'
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

# Nginx Container
resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "terraform-nginx"

  ports {
    internal = 80
    external = 8080
  }

  volumes {
    host_path      = "${path.cwd}/html"
    container_path = "/usr/share/nginx/html"
  }
}

# MariaDB Container
resource "docker_image" "mariadb" {
  name         = "mariadb:latest"
  keep_locally = false
}

resource "docker_container" "mariadb" {
  image = docker_image.mariadb.image_id
  name  = "terraform-mariadb"

  ports {
    internal = 3306
    external = 3306
  }

  env = [
    "MYSQL_ROOT_PASSWORD=rootpass",
    "MYSQL_DATABASE=testdb",
    "MYSQL_USER=testuser",
    "MYSQL_PASSWORD=testpass"
  ]

  volumes {
    volume_name    = docker_volume.mariadb_data.name
    container_path = "/var/lib/mysql"
  }
}

# Volume untuk MariaDB
resource "docker_volume" "mariadb_data" {
  name = "terraform-mariadb-data"
}

# Network
resource "docker_network" "app_network" {
  name = "terraform-app-network"
}

# Output
output "nginx_url" {
  value = "http://localhost:${docker_container.nginx.ports[0].external}"
}
EOF
```

### Langkah 4: Buat variables.tf
```bash
cat > variables.tf << 'EOF'
variable "nginx_port" {
  description = "External port for Nginx"
  type        = number
  default     = 8080
}

variable "db_root_password" {
  description = "MariaDB root password"
  type        = string
  default     = "rootpass"
  sensitive   = true
}
EOF
```

### Langkah 5: Jalankan Terraform
```bash
# Initialize Terraform
terraform init

# Lihat plan
terraform plan

# Apply configuration
terraform apply

# Lihat state
terraform show

# Destroy infrastructure
terraform destroy
```

---

## Ansible untuk Docker

Ansible memungkinkan automation dan configuration management.

### Langkah 1: Install Ansible
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### Langkah 2: Buat Direktori Ansible
```bash
mkdir -p ~/docker-projects/ansible
cd ~/docker-projects/ansible
```

### Langkah 3: Buat Inventory File
```bash
cat > inventory.ini << 'EOF'
[local]
localhost ansible_connection=local

[docker_hosts]
localhost ansible_connection=local
EOF
```

### Langkah 4: Buat Ansible Playbook
```bash
cat > docker-setup.yml << 'EOF'
---
- name: Setup Docker Containers
  hosts: local
  become: yes
  tasks:
    - name: Install required packages
      apt:
        name:
          - docker.io
          - docker-compose
          - python3-pip
        state: present
        update_cache: yes

    - name: Install Docker Python module
      pip:
        name: docker
        state: present

    - name: Ensure Docker service is running
      service:
        name: docker
        state: started
        enabled: yes

    - name: Create Docker network
      docker_network:
        name: ansible-network
        state: present

    - name: Deploy Nginx container
      docker_container:
        name: ansible-nginx
        image: nginx:latest
        state: started
        restart_policy: unless-stopped
        ports:
          - "8090:80"
        networks:
          - name: ansible-network

    - name: Deploy MariaDB container
      docker_container:
        name: ansible-mariadb
        image: mariadb:latest
        state: started
        restart_policy: unless-stopped
        env:
          MYSQL_ROOT_PASSWORD: "ansible_root_pass"
          MYSQL_DATABASE: "ansible_db"
          MYSQL_USER: "ansible_user"
          MYSQL_PASSWORD: "ansible_pass"
        ports:
          - "3307:3306"
        volumes:
          - ansible-mariadb-data:/var/lib/mysql
        networks:
          - name: ansible-network

    - name: Deploy Python Flask container
      docker_container:
        name: ansible-python
        image: python:3.11-slim
        state: started
        command: python3 -m http.server 5001
        ports:
          - "5001:5001"
        networks:
          - name: ansible-network
EOF
```

### Langkah 5: Buat Playbook untuk Management
```bash
cat > docker-manage.yml << 'EOF'
---
- name: Manage Docker Containers
  hosts: local
  become: yes
  vars_prompt:
    - name: action
      prompt: "Enter action (start/stop/restart/remove)"
      private: no
  tasks:
    - name: Manage containers
      docker_container:
        name: "{{ item }}"
        state: "{{ action }}ed"
      loop:
        - ansible-nginx
        - ansible-mariadb
        - ansible-python
      when: action in ['start', 'stop', 'restart']

    - name: Remove containers
      docker_container:
        name: "{{ item }}"
        state: absent
      loop:
        - ansible-nginx
        - ansible-mariadb
        - ansible-python
      when: action == 'remove'
EOF
```

### Langkah 6: Buat ansible.cfg
```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False
EOF
```

### Langkah 7: Jalankan Ansible Playbook
```bash
# Setup containers
ansible-playbook docker-setup.yml

# Manage containers
ansible-playbook docker-manage.yml

# Check syntax
ansible-playbook docker-setup.yml --syntax-check

# Dry run
ansible-playbook docker-setup.yml --check
```

---

## CI/CD Pipeline

### GitLab CI/CD

#### Langkah 1: Buat .gitlab-ci.yml
```bash
cat > .gitlab-ci.yml << 'EOF'
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

before_script:
  - docker info

# Build Stage
build_nginx:
  stage: build
  script:
    - docker build -t my-nginx:$CI_COMMIT_SHORT_SHA ./nginx
    - docker tag my-nginx:$CI_COMMIT_SHORT_SHA my-nginx:latest
  only:
    - main
    - develop

build_php:
  stage: build
  script:
    - docker build -t my-php:$CI_COMMIT_SHORT_SHA ./php
    - docker tag my-php:$CI_COMMIT_SHORT_SHA my-php:latest
  only:
    - main

build_python:
  stage: build
  script:
    - docker build -t my-python:$CI_COMMIT_SHORT_SHA ./python
    - docker tag my-python:$CI_COMMIT_SHORT_SHA my-python:latest
  only:
    - main

build_nodejs:
  stage: build
  script:
    - docker build -t my-nodejs:$CI_COMMIT_SHORT_SHA ./nodejs
    - docker tag my-nodejs:$CI_COMMIT_SHORT_SHA my-nodejs:latest
  only:
    - main

# Test Stage
test_containers:
  stage: test
  script:
    - docker compose -f docker-compose.test.yml up -d
    - sleep 10
    - curl -f http://nginx || exit 1
    - docker compose -f docker-compose.test.yml down
  only:
    - main

# Deploy Stage
deploy_production:
  stage: deploy
  script:
    - docker compose -f docker-compose.prod.yml pull
    - docker compose -f docker-compose.prod.yml up -d
  only:
    - main
  when: manual
EOF
```

### GitHub Actions

#### Langkah 2: Buat .github/workflows/docker.yml
```bash
mkdir -p .github/workflows
cat > .github/workflows/docker.yml << 'EOF'
name: Docker CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Build Nginx image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/my-nginx:${{ github.sha }} ./nginx
        docker tag ${{ secrets.DOCKER_USERNAME }}/my-nginx:${{ github.sha }} ${{ secrets.DOCKER_USERNAME }}/my-nginx:latest
    
    - name: Build PHP image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/my-php:${{ github.sha }} ./php
        docker tag ${{ secrets.DOCKER_USERNAME }}/my-php:${{ github.sha }} ${{ secrets.DOCKER_USERNAME }}/my-php:latest
    
    - name: Build Python image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/my-python:${{ github.sha }} ./python
        docker tag ${{ secrets.DOCKER_USERNAME }}/my-python:${{ github.sha }} ${{ secrets.DOCKER_USERNAME }}/my-python:latest
    
    - name: Build Node.js image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/my-nodejs:${{ github.sha }} ./nodejs
        docker tag ${{ secrets.DOCKER_USERNAME }}/my-nodejs:${{ github.sha }} ${{ secrets.DOCKER_USERNAME }}/my-nodejs:latest
    
    - name: Run tests
      run: |
        docker compose -f docker-compose.test.yml up -d
        sleep 15
        curl -f http://localhost:80 || exit 1
        docker compose -f docker-compose.test.yml down
    
    - name: Push images to Docker Hub
      if: github.ref == 'refs/heads/main'
      run: |
        docker push ${{ secrets.DOCKER_USERNAME }}/my-nginx:${{ github.sha }}
        docker push ${{ secrets.DOCKER_USERNAME }}/my-nginx:latest
        docker push ${{ secrets.DOCKER_USERNAME }}/my-php:${{ github.sha }}
        docker push ${{ secrets.DOCKER_USERNAME }}/my-php:latest
        docker push ${{ secrets.DOCKER_USERNAME }}/my-python:${{ github.sha }}
        docker push ${{ secrets.DOCKER_USERNAME }}/my-python:latest
        docker push ${{ secrets.DOCKER_USERNAME }}/my-nodejs:${{ github.sha }}
        docker push ${{ secrets.DOCKER_USERNAME }}/my-nodejs:latest

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Deploy to production
      run: |
        # Deploy menggunakan SSH atau deployment tool
        echo "Deploying to production server..."
        # ssh user@server "cd /app && docker compose pull && docker compose up -d"
EOF
```

### Jenkins Pipeline

#### Langkah 3: Buat Jenkinsfile
```bash
cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Images') {
            parallel {
                stage('Build Nginx') {
                    steps {
                        script {
                            docker.build("my-nginx:${IMAGE_TAG}", "./nginx")
                        }
                    }
                }
                stage('Build PHP') {
                    steps {
                        script {
                            docker.build("my-php:${IMAGE_TAG}", "./php")
                        }
                    }
                }
                stage('Build Python') {
                    steps {
                        script {
                            docker.build("my-python:${IMAGE_TAG}", "./python")
                        }
                    }
                }
                stage('Build Node.js') {
                    steps {
                        script {
                            docker.build("my-nodejs:${IMAGE_TAG}", "./nodejs")
                        }
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                sh 'docker compose -f docker-compose.test.yml up -d'
                sh 'sleep 15'
                sh 'curl -f http://localhost:80 || exit 1'
                sh 'docker compose -f docker-compose.test.yml down'
            }
        }
        
        stage('Push to Registry') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-hub-credentials') {
                        docker.image("my-nginx:${IMAGE_TAG}").push()
                        docker.image("my-nginx:${IMAGE_TAG}").push('latest')
                        docker.image("my-php:${IMAGE_TAG}").push()
                        docker.image("my-python:${IMAGE_TAG}").push()
                        docker.image("my-nodejs:${IMAGE_TAG}").push()
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    docker compose -f docker-compose.prod.yml pull
                    docker compose -f docker-compose.prod.yml up -d
                '''
            }
        }
    }
    
    post {
        always {
            sh 'docker compose -f docker-compose.test.yml down || true'
            cleanWs()
        }
        success {
            echo 'Pipeline berhasil!'
        }
        failure {
            echo 'Pipeline gagal!'
        }
    }
}
EOF
```

### Docker Compose untuk Testing

#### Langkah 4: Buat docker-compose.test.yml
```bash
cat > docker-compose.test.yml << 'EOF'
version: '3.8'

services:
  nginx:
    build: ./nginx
    ports:
      - "80:80"
    networks:
      - test-network

  mariadb:
    image: mariadb:latest
    environment:
      MYSQL_ROOT_PASSWORD: testpass
      MYSQL_DATABASE: testdb
    networks:
      - test-network

  php:
    build: ./php
    networks:
      - test-network

  python:
    build: ./python
    networks:
      - test-network

  nodejs:
    build: ./nodejs
    networks:
      - test-network

networks:
  test-network:
    driver: bridge
EOF
```

### Docker Compose untuk Production

#### Langkah 5: Buat docker-compose.prod.yml
```bash
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: ${DOCKER_USERNAME}/my-nginx:latest
    container_name: prod-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/ssl:/etc/nginx/ssl:ro
    networks:
      - prod-network
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  mariadb:
    image: mariadb:latest
    container_name: prod-mariadb
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mariadb-prod-data:/var/lib/mysql
    networks:
      - prod-network
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  php:
    image: ${DOCKER_USERNAME}/my-php:latest
    container_name: prod-php
    networks:
      - prod-network
    restart: always

  python:
    image: ${DOCKER_USERNAME}/my-python:latest
    container_name: prod-python
    networks:
      - prod-network
    restart: always

  nodejs:
    image: ${DOCKER_USERNAME}/my-nodejs:latest
    container_name: prod-nodejs
    networks:
      - prod-network
    restart: always

volumes:
  mariadb-prod-data:

networks:
  prod-network:
    driver: bridge
EOF
```

---

## Tips dan Best Practices

### 1. Security Best Practices
```bash
# Jangan gunakan root user di container
# Gunakan specific user dalam Dockerfile
RUN useradd -m -u 1000 appuser
USER appuser

# Scan image untuk vulnerabilities
docker scan nama-image:tag

# Gunakan secrets untuk sensitive data
docker secret create db_password ./password.txt

# Update images secara berkala
docker pull image:latest
```

### 2. Optimasi Image Size
```bash
# Gunakan multi-stage builds
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]

# Gunakan .dockerignore
echo "node_modules" >> .dockerignore
echo ".git" >> .dockerignore
echo "*.md" >> .dockerignore
```

### 3. Monitoring dan Logging
```bash
# Lihat resource usage
docker stats

# Export logs
docker logs container-name > app.log

# Gunakan log driver
docker run --log-driver=syslog nginx

# Monitoring dengan cAdvisor
docker run -d \
  --name=cadvisor \
  -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  google/cadvisor:latest
```

### 4. Backup dan Recovery
```bash
# Backup container
docker commit container-name backup-image:tag

# Export container
docker export container-name > backup.tar

# Import container
docker import backup.tar restored-image:tag

# Backup volumes
docker run --rm \
  -v volume-name:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/backup.tar.gz /data
```

### 5. Performance Tuning
```bash
# Limit resources
docker run -d \
  --memory="512m" \
  --cpus="1.5" \
  --name limited-container \
  nginx

# Prune unused resources
docker system prune -a

# Clean build cache
docker builder prune
```

---

## Troubleshooting

### Problem: Container tidak bisa start
```bash
# Check logs
docker logs container-name

# Check events
docker events

# Inspect container
docker inspect container-name
```

### Problem: Port already in use
```bash
# Find process using port
sudo lsof -i :8080

# Kill process
sudo kill -9 PID

# Atau gunakan port lain
docker run -p 8081:80 nginx
```

### Problem: Permission denied
```bash
# Fix Docker socket permission
sudo chmod 666 /var/run/docker.sock

# Atau tambahkan user ke docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Problem: Disk space full
```bash
# Check disk usage
docker system df

# Clean up
docker system prune -a --volumes

# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune
```

### Problem: Network issues
```bash
# List networks
docker network ls

# Inspect network
docker network inspect network-name

# Remove network
docker network rm network-name

# Recreate network
docker network create network-name
```

---

## Referensi dan Resources

### Dokumentasi Official
- Docker: https://docs.docker.com
- Docker Compose: https://docs.docker.com/compose
- Terraform: https://www.terraform.io/docs
- Ansible: https://docs.ansible.com

### Tool Tambahan
- Portainer: Web UI untuk manage Docker
- Lazydocker: Terminal UI untuk Docker
- Dive: Tool untuk analyze image layers
- Hadolint: Dockerfile linter

### Perintah Install Tool Tambahan
```bash
# Portainer
docker run -d \
  -p 9000:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce

# Lazydocker
curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh | bash

# Dive
wget https://github.com/wagoodman/dive/releases/download/v0.11.0/dive_0.11.0_linux_amd64.deb
sudo dpkg -i dive_0.11.0_linux_amd64.deb
```

---

## Kesimpulan

Anda telah mempelajari:
1. ✅ Instalasi dan konfigurasi Docker
2. ✅ Membuat container untuk berbagai services (Nginx, MariaDB, PHP, Python, Node.js)
3. ✅ Menggunakan Docker Compose untuk multi-container apps
4. ✅ Infrastructure as Code dengan Terraform
5. ✅ Automation dengan Ansible
6. ✅ CI/CD pipeline dengan GitLab, GitHub Actions, dan Jenkins

### Next Steps
- Pelajari Kubernetes untuk orchestration lebih advanced
- Implementasi monitoring dengan Prometheus & Grafana
- Setup reverse proxy dengan Traefik
- Implementasi service mesh dengan Istio
- Eksplorasi Docker Swarm untuk clustering

Selamat belajar Docker! 🐳