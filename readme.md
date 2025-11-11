# DevOps Project: Video-to-Audio Converter на Proxmox
<img width="741" height="481" alt="image" src="https://github.com/user-attachments/assets/de7d299c-8ac9-4ee5-bbe8-b83f035a1af0" />

## Описание проекта
 
Микросервисное приложение для конвертации видео (MP4) в аудио (MP3) на базе Python и Kubernetes, адаптированное для развертывания в домашней инфраструктуре Proxmox.

## Архитектура решения

### Компоненты системы

- **K3s Cluster** — легковесный Kubernetes вместо AWS EKS
- **MinIO** — S3-совместимое хранилище вместо AWS S3
- **PostgreSQL** — база данных для auth-service
- **MongoDB** — база данных для converter-service
- **RabbitMQ** — брокер сообщений для очередей
- **Ingress Controller** — для маршрутизации трафика
- **MetalLB** — балансировщик нагрузки для bare-metal
- **Cloudflare Tunnel** — для внешнего доступа (альтернатива ngrok)

### Микросервисы приложения

1. **auth-service** — аутентификация и выдача JWT токенов
2. **gateway-service** — API Gateway для обработки запросов
3. **converter-service** — конвертация видео в аудио
4. **notification-service** — отправка email уведомлений

## Предварительные требования

### Установленное ПО на Windows 10

```powershell
# Chocolatey (менеджер пакетов для Windows)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Установка необходимых инструментов
choco install terraform -y
choco install kubectl -y
choco install helm -y
choco install git -y
choco install putty -y
choco install python -y
```

### SSH ключи для доступа к VM

```powershell
# Создание SSH ключа
ssh-keygen -t ed25519 -C "proxmox-cluster" -f $HOME\.ssh\proxmox_cluster
```

## Планирование инфраструктуры

### Виртуальные машины

| Hostname | IP | CPU | RAM | Disk | Роль |
|----------|------------|-----|-----|------|------|
| k3s-master | 10.0.10.51 | 4 | 8GB | 50GB | K3s Control Plane |
| k3s-worker1 | 10.0.10.52 | 4 | 8GB | 50GB | K3s Worker Node |
| k3s-worker2 | 10.0.10.53 | 4 | 8GB | 50GB | K3s Worker Node |
| minio-server | 10.0.10.54 | 2 | 4GB | 100GB | Object Storage |
| postgres-db | 10.0.10.55 | 2 | 4GB | 30GB | PostgreSQL Database |
| mongo-db | 10.0.10.56 | 2 | 4GB | 30GB | MongoDB Database |

**Итого:** 16 CPU, 36GB RAM, 310GB Disk

### Сетевая конфигурация

- **Внутренняя сеть:** 10.0.10.0/24
- **Gateway:** 10.0.10.1 (TP-Link router)
- **Proxmox Host:** 10.0.10.200
- **MetalLB IP Pool:** 10.0.10.100-10.0.10.110
- **DNS:** 8.8.8.8, 1.1.1.1 (публичные)

## Этап 1: Создание VM через Terraform

### Структура Terraform проекта

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── templates/
│   └── cloud-init.yaml
└── terraform.tfvars
```

### variables.tf

```hcl
variable "proxmox_host" {
  description = "Proxmox host address"
  type        = string
  default     = "10.0.10.200"
}

variable "proxmox_port" {
  description = "Proxmox API port"
  type        = string
  default     = "8006"
}

variable "proxmox_api_token_id" {
  description = "Proxmox API Token ID"
  type        = string
  sensitive   = true
}

variable "proxmox_api_token_secret" {
  description = "Proxmox API Token Secret"
  type        = string
  sensitive   = true
}

variable "ssh_public_key" {
  description = "SSH public key for VM access"
  type        = string
}

variable "gateway" {
  description = "Network gateway"
  type        = string
  default     = "10.0.10.1"
}

variable "nameservers" {
  description = "DNS servers"
  type        = string
  default     = "8.8.8.8 1.1.1.1"
}

variable "vms" {
  description = "Virtual machines configuration"
  type = map(object({
    ip     = string
    cores  = number
    memory = number
    disk   = string
  }))
  default = {
    "k3s-master" = {
      ip     = "10.0.10.51"
      cores  = 4
      memory = 8192
      disk   = "50G"
    }
    "k3s-worker1" = {
      ip     = "10.0.10.52"
      cores  = 4
      memory = 8192
      disk   = "50G"
    }
    "k3s-worker2" = {
      ip     = "10.0.10.53"
      cores  = 4
      memory = 8192
      disk   = "50G"
    }
    "minio-server" = {
      ip     = "10.0.10.54"
      cores  = 2
      memory = 4096
      disk   = "100G"
    }
    "postgres-db" = {
      ip     = "10.0.10.55"
      cores  = 2
      memory = 4096
      disk   = "30G"
    }
    "mongo-db" = {
      ip     = "10.0.10.56"
      cores  = 2
      memory = 4096
      disk   = "30G"
    }
  }
}
```

### main.tf

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

provider "proxmox" {
  pm_api_url          = "https://${var.proxmox_host}:${var.proxmox_port}/api2/json"
  pm_api_token_id     = var.proxmox_api_token_id
  pm_api_token_secret = var.proxmox_api_token_secret
  pm_tls_insecure     = true
}

resource "proxmox_vm_qemu" "vms" {
  for_each = var.vms

  name        = each.key
  target_node = "pve"  # Измените на имя вашего Proxmox узла
  clone       = "ubuntu-22-04-template"  # Имя вашего template
  
  cores   = each.value.cores
  memory  = each.value.memory
  sockets = 1
  
  disk {
    size    = each.value.disk
    type    = "scsi"
    storage = "local-lvm"  # Измените на ваше хранилище
  }

  network {
    model  = "virtio"
    bridge = "vmbr0"
  }

  ipconfig0 = "ip=${each.value.ip}/24,gw=${var.gateway}"
  
  nameserver = var.nameservers

  sshkeys = var.ssh_public_key

  os_type   = "cloud-init"
  ciuser    = "ubuntu"
  
  agent = 1

  lifecycle {
    ignore_changes = [
      network,
    ]
  }
}
```

### terraform.tfvars

```hcl
proxmox_api_token_id     = "root@pam!terraform"
proxmox_api_token_secret = "ваш-секретный-токен"
ssh_public_key           = "ssh-ed25519 AAAA... ваш-публичный-ключ"
```

### Создание API токена в Proxmox

```bash
# Подключитесь к Proxmox через SSH
ssh root@10.0.10.200

# Создайте API токен
pveum user token add root@pam terraform -privsep 0
```

### Создание Ubuntu Cloud-Init Template

```bash
# На Proxmox хосте
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Создание template
qm create 9000 --name ubuntu-22-04-template --memory 2048 --net0 virtio,bridge=vmbr0
qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
qm template 9000
```

### Развертывание VM

```powershell
# На Windows машине
cd C:\Projects\video-converter\terraform

# Инициализация
terraform init

# Проверка плана
terraform plan

# Применение конфигурации
terraform apply -auto-approve

# Получение IP адресов
terraform output
```

### Проверка доступа к VM

```powershell
# Тест SSH подключения
ssh -i $HOME\.ssh\proxmox_cluster ubuntu@10.0.10.51

# Проверка всех VM
$vms = @("10.0.10.51", "10.0.10.52", "10.0.10.53", "10.0.10.54", "10.0.10.55", "10.0.10.56")
foreach ($vm in $vms) {
    Write-Host "Testing $vm..." -ForegroundColor Cyan
    ssh -i $HOME\.ssh\proxmox_cluster -o ConnectTimeout=5 ubuntu@$vm "hostname && uptime"
}
```

## Этап 2: Настройка баз данных

### PostgreSQL на postgres-db (10.0.10.55)

```bash
# Подключение к VM
ssh ubuntu@10.0.10.55

# Установка PostgreSQL
sudo apt update
sudo apt install -y postgresql postgresql-contrib

# Настройка PostgreSQL
sudo -u postgres psql

-- В psql консоли
CREATE USER vidconv WITH PASSWORD 'StrongPassword123!';
CREATE DATABASE authdb OWNER vidconv;
\c authdb

-- Создание таблицы users
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL
);

-- Выход
\q

# Настройка удаленного доступа
sudo nano /etc/postgresql/14/main/postgresql.conf
# Изменить: listen_addresses = '*'

sudo nano /etc/postgresql/14/main/pg_hba.conf
# Добавить: host    all             all             10.0.10.0/24            md5

# Перезапуск PostgreSQL
sudo systemctl restart postgresql
sudo systemctl enable postgresql
```

### Проверка подключения к PostgreSQL

```powershell
# С Windows машины
psql "postgresql://vidconv:StrongPassword123!@10.0.10.55:5432/authdb"
```

### MongoDB на mongo-db (10.0.10.56)

```bash
# Подключение к VM
ssh ubuntu@10.0.10.56

# Установка MongoDB
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt update
sudo apt install -y mongodb-org

# Запуск MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod

# Создание пользователя
mongosh

use admin
db.createUser({
  user: "vidconv",
  pwd: "StrongPassword123!",
  roles: [ { role: "root", db: "admin" } ]
})

use mp3s
db.createCollection("fs.files")
db.createCollection("fs.chunks")

exit

# Настройка удаленного доступа
sudo nano /etc/mongod.conf
# Изменить:
# net:
#   bindIp: 0.0.0.0

# Включить аутентификацию:
# security:
#   authorization: enabled

# Перезапуск MongoDB
sudo systemctl restart mongod
```

### Проверка подключения к MongoDB

```powershell
# С Windows машины (установите mongosh)
mongosh "mongodb://vidconv:StrongPassword123!@10.0.10.56:27017/mp3s?authSource=admin"
```

## Этап 3: Настройка MinIO

### Установка MinIO на minio-server (10.0.10.54)

```bash
# Подключение к VM
ssh ubuntu@10.0.10.54

# Установка MinIO
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# Создание директории для данных
sudo mkdir -p /mnt/minio-data
sudo chown ubuntu:ubuntu /mnt/minio-data

# Создание systemd service
sudo nano /etc/systemd/system/minio.service
```

### Содержимое minio.service

```ini
[Unit]
Description=MinIO
Documentation=https://min.io/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local

User=ubuntu
Group=ubuntu

Environment="MINIO_ROOT_USER=minioadmin"
Environment="MINIO_ROOT_PASSWORD=MinioPassword123!"
Environment="MINIO_VOLUMES=/mnt/minio-data"
Environment="MINIO_OPTS=--console-address :9001"

ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

Restart=always
LimitNOFILE=65536
TasksMax=infinity

[Install]
WantedBy=multi-user.target
```

### Запуск MinIO

```bash
# Запуск и включение автозапуска
sudo systemctl daemon-reload
sudo systemctl start minio
sudo systemctl enable minio

# Проверка статуса
sudo systemctl status minio
```

### Создание бакета в MinIO

```bash
# Установка MinIO Client
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Настройка alias
mc alias set myminio http://10.0.10.54:9000 minioadmin MinioPassword123!

# Создание бакетов
mc mb myminio/videos
mc mb myminio/mp3s

# Проверка
mc ls myminio
```

### Доступ к MinIO Console

Откройте браузер: `http://10.0.10.54:9001`
- Username: `minioadmin`
- Password: `MinioPassword123!`

## Этап 4: Установка K3s кластера

### Установка K3s Master на k3s-master (10.0.10.51)

```bash
# Подключение к VM
ssh ubuntu@10.0.10.51

# Установка K3s без traefik (используем свой ingress)
curl -sfL https://get.k3s.io | sh -s - server \
  --disable traefik \
  --write-kubeconfig-mode 644 \
  --node-ip 10.0.10.51 \
  --advertise-address 10.0.10.51 \
  --tls-san 10.0.10.51

# Получение токена для worker nodes
sudo cat /var/lib/rancher/k3s/server/node-token

# Копирование kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown ubuntu:ubuntu ~/.kube/config
```

### Копирование kubeconfig на Windows

```powershell
# На Windows машине
scp ubuntu@10.0.10.51:~/.kube/config $HOME\.kube\config-proxmox

# Редактирование config
# Измените server: https://127.0.0.1:6443 на server: https://10.0.10.51:6443

# Установка переменной окружения
$env:KUBECONFIG="$HOME\.kube\config-proxmox"

# Проверка подключения
kubectl get nodes
```

### Присоединение Worker Nodes

```bash
# На k3s-worker1 (10.0.10.52)
ssh ubuntu@10.0.10.52

curl -sfL https://get.k3s.io | K3S_URL=https://10.0.10.51:6443 \
  K3S_TOKEN=<токен-с-master-ноды> \
  sh -s - agent --node-ip 10.0.10.52

# На k3s-worker2 (10.0.10.53)
ssh ubuntu@10.0.10.53

curl -sfL https://get.k3s.io | K3S_URL=https://10.0.10.51:6443 \
  K3S_TOKEN=<токен-с-master-ноды> \
  sh -s - agent --node-ip 10.0.10.53
```

### Проверка кластера

```powershell
# На Windows машине
kubectl get nodes
kubectl get pods -A
```

## Этап 5: Установка MetalLB

### Установка MetalLB через манифесты

```powershell
# Установка MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Ожидание готовности
kubectl wait --namespace metallb-system `
  --for=condition=ready pod `
  --selector=app=metallb `
  --timeout=90s
```

### Конфигурация IP Pool

```yaml
# metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.0.10.100-10.0.10.110
```

```powershell
kubectl apply -f metallb-config.yaml
```

## Этап 6: Установка Ingress Controller

### Установка NGINX Ingress Controller

```powershell
# Установка через Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx `
  --namespace ingress-nginx `
  --create-namespace `
  --set controller.service.type=LoadBalancer `
  --set controller.service.externalTrafficPolicy=Local

# Проверка
kubectl get svc -n ingress-nginx
```

### Получение External IP

```powershell
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Запомните EXTERNAL-IP (например, 10.0.10.100)
```

## Этап 7: Установка RabbitMQ

### Создание Namespace

```powershell
kubectl create namespace rabbitmq
```

### Установка RabbitMQ через Helm

```powershell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install rabbitmq bitnami/rabbitmq `
  --namespace rabbitmq `
  --set auth.username=guest `
  --set auth.password=guest `
  --set service.type=NodePort `
  --set service.nodePorts.amqp=30672 `
  --set service.nodePorts.manager=30673

# Проверка
kubectl get pods -n rabbitmq
kubectl get svc -n rabbitmq
```

### Создание очередей в RabbitMQ

```powershell
# Получение пароля
$RABBITMQ_PASSWORD = kubectl get secret --namespace rabbitmq rabbitmq -o jsonpath="{.data.rabbitmq-password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Port forwarding для доступа
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672

# Откройте браузер: http://localhost:15672
# Username: guest
# Password: (используйте $RABBITMQ_PASSWORD)
```

Создайте две очереди в интерфейсе RabbitMQ:
- `video` (для видео файлов)
- `mp3` (для аудио файлов)

## Этап 8: Подготовка манифестов приложения

### Структура проекта

```
video-converter/
├── auth-service/
│   └── manifests/
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── secret.yaml
│       └── service.yaml
├── gateway-service/
│   └── manifests/
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── secret.yaml
│       └── service.yaml
├── converter-service/
│   └── manifests/
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── secret.yaml
│       └── service.yaml
├── notification-service/
│   └── manifests/
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── secret.yaml
│       └── service.yaml
└── ingress.yaml
```

### auth-service/manifests/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-configmap
  namespace: default
data:
  POSTGRES_HOST: "10.0.10.55"
  POSTGRES_PORT: "5432"
  POSTGRES_DB: "authdb"
```

### auth-service/manifests/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-secret
  namespace: default
type: Opaque
stringData:
  POSTGRES_USER: "vidconv"
  POSTGRES_PASSWORD: "StrongPassword123!"
  JWT_SECRET: "your-super-secret-jwt-key-change-this"
```

### auth-service/manifests/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: your-dockerhub-username/auth-service:latest
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: auth-configmap
        - secretRef:
            name: auth-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

### auth-service/manifests/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: default
spec:
  selector:
    app: auth-service
  ports:
  - port: 5000
    targetPort: 5000
  type: ClusterIP
```

### gateway-service/manifests/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gateway-configmap
  namespace: default
data:
  AUTH_SERVICE_HOST: "auth-service"
  AUTH_SERVICE_PORT: "5000"
  MINIO_ENDPOINT: "10.0.10.54:9000"
  MINIO_BUCKET_VIDEOS: "videos"
  MINIO_BUCKET_MP3S: "mp3s"
  MONGODB_HOST: "10.0.10.56"
  MONGODB_PORT: "27017"
  MONGODB_DATABASE: "mp3s"
  RABBITMQ_HOST: "rabbitmq.rabbitmq.svc.cluster.local"
  RABBITMQ_PORT: "5672"
  RABBITMQ_VIDEO_QUEUE: "video"
  RABBITMQ_MP3_QUEUE: "mp3"
```

### gateway-service/manifests/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gateway-secret
  namespace: default
type: Opaque
stringData:
  MINIO_ACCESS_KEY: "minioadmin"
  MINIO_SECRET_KEY: "MinioPassword123!"
  MONGODB_USERNAME: "vidconv"
  MONGODB_PASSWORD: "StrongPassword123!"
  RABBITMQ_USERNAME: "guest"
  RABBITMQ_PASSWORD: "guest"
```

### gateway-service/manifests/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gateway-service
  template:
    metadata:
      labels:
        app: gateway-service
    spec:
      containers:
      - name: gateway-service
        image: your-dockerhub-username/gateway-service:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: gateway-configmap
        - secretRef:
            name: gateway-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### gateway-service/manifests/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gateway-service
  namespace: default
spec:
  selector:
    app: gateway-service
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

### converter-service/manifests/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: converter-configmap
  namespace: default
data:
  MINIO_ENDPOINT: "10.0.10.54:9000"
  MINIO_BUCKET_VIDEOS: "videos"
  MINIO_BUCKET_MP3S: "mp3s"
  MONGODB_HOST: "10.0.10.56"
  MONGODB_PORT: "27017"
  MONGODB_DATABASE: "mp3s"
  RABBITMQ_HOST: "rabbitmq.rabbitmq.svc.cluster.local"
  RABBITMQ_PORT: "5672"
  RABBITMQ_VIDEO_QUEUE: "video"
  RABBITMQ_MP3_QUEUE: "mp3"
```

### converter-service/manifests/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: converter-secret
  namespace: default
type: Opaque
stringData:
  MINIO_ACCESS_KEY: "minioadmin"
  MINIO_SECRET_KEY: "MinioPassword123!"
  MONGODB_USERNAME: "vidconv"
  MONGODB_PASSWORD: "StrongPassword123!"
  RABBITMQ_USERNAME: "guest"
  RABBITMQ_PASSWORD: "guest"
```

### converter-service/manifests/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: converter-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: converter-service
  template:
    metadata:
      labels:
        app: converter-service
    spec:
      containers:
      - name: converter-service
        image: your-dockerhub-username/converter-service:latest
        envFrom:
        - configMapRef:
            name: converter-configmap
        - secretRef:
            name: converter-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

### notification-service/manifests/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notification-configmap
  namespace: default
data:
  RABBITMQ_HOST: "rabbitmq.rabbitmq.svc.cluster.local"
  RABBITMQ_PORT: "5672"
  RABBITMQ_MP3_QUEUE: "mp3"
  SMTP_SERVER: "smtp.gmail.com"
  SMTP_PORT: "587"
```

### notification-service/manifests/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: notification-secret
  namespace: default
type: Opaque
stringData:
  RABBITMQ_USERNAME: "guest"
  RABBITMQ_PASSWORD: "guest"
  GMAIL_ADDRESS: "your-email@gmail.com"
  GMAIL_PASSWORD: "your-app-specific-password"  # Получите в настройках Google
```

### notification-service/manifests/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
    spec:
      containers:
      - name: notification-service
        image: your-dockerhub-username/notification-service:latest
        envFrom:
        - configMapRef:
            name: notification-configmap
        - secretRef:
            name: notification-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: video-converter-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"
spec:
  ingressClassName: nginx
  rules:
  - host: video-converter.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway-service
            port:
              number: 8080
```

## Этап 9: Сборка Docker образов

### Клонирование репозитория проекта

```powershell
# На Windows машине
git clone https://github.com/original-repo/video-converter.git
cd video-converter
```

### Создание Dockerfile для каждого сервиса

### auth-service/Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "server.py"]
```

### auth-service/requirements.txt

```
Flask==2.3.0
psycopg2-binary==2.9.6
PyJWT==2.8.0
bcrypt==4.0.1
```

### gateway-service/Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["python", "server.py"]
```

### gateway-service/requirements.txt

```
Flask==2.3.0
PyJWT==2.8.0
pymongo==4.4.0
pika==1.3.2
minio==7.1.15
gridfs==4.0.0
requests==2.31.0
```

### converter-service/Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Установка ffmpeg
RUN apt-get update && \
    apt-get install -y ffmpeg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "consumer.py"]
```

### converter-service/requirements.txt

```
pika==1.3.2
pymongo==4.4.0
minio==7.1.15
gridfs==4.0.0
```

### notification-service/Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "consumer.py"]
```

### notification-service/requirements.txt

```
pika==1.3.2
smtplib  # Встроен в Python
email  # Встроен в Python
```

### Сборка и публикация образов

```powershell
# Логин в Docker Hub
docker login

# Сборка образов
docker build -t your-dockerhub-username/auth-service:latest ./auth-service
docker build -t your-dockerhub-username/gateway-service:latest ./gateway-service
docker build -t your-dockerhub-username/converter-service:latest ./converter-service
docker build -t your-dockerhub-username/notification-service:latest ./notification-service

# Публикация образов
docker push your-dockerhub-username/auth-service:latest
docker push your-dockerhub-username/gateway-service:latest
docker push your-dockerhub-username/converter-service:latest
docker push your-dockerhub-username/notification-service:latest
```

## Этап 10: Развертывание приложения

### Применение манифестов

```powershell
# Установка переменной окружения
$env:KUBECONFIG="$HOME\.kube\config-proxmox"

# Развертывание auth-service
kubectl apply -f auth-service/manifests/

# Развертывание gateway-service
kubectl apply -f gateway-service/manifests/

# Развертывание converter-service
kubectl apply -f converter-service/manifests/

# Развертывание notification-service
kubectl apply -f notification-service/manifests/

# Развертывание Ingress
kubectl apply -f ingress.yaml

# Проверка статуса
kubectl get pods
kubectl get svc
kubectl get ingress
```

### Проверка логов

```powershell
# Просмотр логов
kubectl logs -l app=auth-service --tail=50
kubectl logs -l app=gateway-service --tail=50
kubectl logs -l app=converter-service --tail=50
kubectl logs -l app=notification-service --tail=50
```

## Этап 11: Настройка внешнего доступа

### Вариант 1: Использование Cloudflare Tunnel (рекомендуется)

```bash
# На k3s-master (10.0.10.51)
ssh ubuntu@10.0.10.51

# Установка cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Аутентификация
cloudflared tunnel login

# Создание туннеля
cloudflared tunnel create video-converter

# Получение Tunnel ID
cloudflared tunnel list

# Создание конфигурации
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

### ~/.cloudflared/config.yml

```yaml
tunnel: <your-tunnel-id>
credentials-file: /home/ubuntu/.cloudflared/<your-tunnel-id>.json

ingress:
  - hostname: video-converter.yourdomain.com
    service: http://10.0.10.100:80
  - service: http_status:404
```

### Настройка DNS в Cloudflare

```bash
# Создание DNS записи
cloudflared tunnel route dns video-converter video-converter.yourdomain.com

# Запуск туннеля
cloudflared tunnel run video-converter

# Создание systemd service для автозапуска
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

### Вариант 2: Использование ngrok

```powershell
# На Windows машине
choco install ngrok -y

# Аутентификация (получите токен на ngrok.com)
ngrok authtoken <your-auth-token>

# Проброс порта к Ingress Controller
ngrok http 10.0.10.100:80
```

### Вариант 3: Port Forwarding на роутере

1. Откройте веб-интерфейс роутера TP-Link (http://10.0.10.1)
2. Найдите раздел "Port Forwarding" или "Virtual Servers"
3. Создайте правило:
   - External Port: 80
   - Internal IP: 10.0.10.100 (MetalLB IP)
   - Internal Port: 80
   - Protocol: TCP
4. Сохраните настройки

### Обновление /etc/hosts для локального доступа

```powershell
# На Windows (запустите PowerShell как Администратор)
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "10.0.10.100 video-converter.local"

# Проверка
ping video-converter.local
```

## Этап 12: Настройка Gmail для уведомлений

### Получение App-Specific Password

1. Откройте https://myaccount.google.com/
2. Перейдите в раздел "Security"
3. Включите "2-Step Verification"
4. Найдите "App passwords"
5. Создайте пароль для приложения "Mail"
6. Скопируйте 16-символьный пароль

### Обновление secret для notification-service

```powershell
# Кодирование пароля в base64
$password = "your-16-char-app-password"
$bytes = [System.Text.Encoding]::UTF8.GetBytes($password)
$encoded = [Convert]::ToBase64String($bytes)
Write-Output $encoded

# Обновление secret
kubectl edit secret notification-secret

# Замените значение GMAIL_PASSWORD на закодированное значение
```

## Этап 13: Создание пользователя для тестирования

### Добавление пользователя в PostgreSQL

```bash
# Подключение к PostgreSQL
psql "postgresql://vidconv:StrongPassword123!@10.0.10.55:5432/authdb"

-- Вставка тестового пользователя
INSERT INTO users (email, password) 
VALUES ('test@example.com', '$2b$12$hashed_password_here');

-- Для создания хэша пароля используйте Python
-- python3 -c "import bcrypt; print(bcrypt.hashpw(b'password123', bcrypt.gensalt()).decode())"
```

### Скрипт для создания пользователя

```python
# create_user.py
import psycopg2
import bcrypt

# Параметры подключения
conn = psycopg2.connect(
    host="10.0.10.55",
    port=5432,
    database="authdb",
    user="vidconv",
    password="StrongPassword123!"
)

# Создание курсора
cur = conn.cursor()

# Данные пользователя
email = "test@example.com"
password = "password123"

# Хэширование пароля
hashed = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')

# Вставка пользователя
cur.execute("INSERT INTO users (email, password) VALUES (%s, %s)", (email, hashed))

# Сохранение изменений
conn.commit()

# Закрытие соединения
cur.close()
conn.close()

print(f"User {email} created successfully!")
```

```powershell
# Запуск скрипта
python create_user.py
```

## Этап 14: Тестирование приложения

### Получение токена аутентификации

```powershell
# Логин
$response = Invoke-WebRequest -Uri "http://video-converter.local/login" `
  -Method POST `
  -Headers @{
    "Authorization" = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("test@example.com:password123"))
  }

$token = $response.Content
Write-Output "JWT Token: $token"
```

### Загрузка видео файла

```powershell
# Загрузка видео
$headers = @{
    "Authorization" = "Bearer $token"
}

$filePath = "C:\path\to\your\video.mp4"

$response = Invoke-WebRequest -Uri "http://video-converter.local/upload" `
  -Method POST `
  -Headers $headers `
  -InFile $filePath `
  -ContentType "multipart/form-data"

Write-Output $response.Content
```

### Проверка email

После загрузки проверьте почту `test@example.com` на наличие письма с file ID (fid).

### Скачивание MP3

```powershell
# Скачивание конвертированного файла
$fid = "file-id-from-email"

$headers = @{
    "Authorization" = "Bearer $token"
}

Invoke-WebRequest -Uri "http://video-converter.local/download?fid=$fid" `
  -Method GET `
  -Headers $headers `
  -OutFile "output.mp3"

Write-Output "File downloaded as output.mp3"
```

### Альтернативное тестирование через curl

```bash
# Логин
curl -X POST http://video-converter.local/login \
  -u test@example.com:password123

# Сохраните токен
TOKEN="your-jwt-token-here"

# Загрузка видео
curl -X POST http://video-converter.local/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/video.mp4"

# Скачивание MP3
curl -X GET "http://video-converter.local/download?fid=file-id" \
  -H "Authorization: Bearer $TOKEN" \
  -o output.mp3
```

## Этап 15: Мониторинг и отладка

### Установка Kubernetes Dashboard

```powershell
# Установка Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Создание ServiceAccount
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard

# Создание ClusterRoleBinding
kubectl create clusterrolebinding dashboard-admin `
  --clusterrole=cluster-admin `
  --serviceaccount=kubernetes-dashboard:dashboard-admin

# Получение токена
kubectl -n kubernetes-dashboard create token dashboard-admin

# Port forwarding
kubectl proxy

# Откройте браузер:
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

### Просмотр логов RabbitMQ

```powershell
# Port forwarding для RabbitMQ Management
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672

# Откройте: http://localhost:15672
# Username: guest, Password: guest
```

### Проверка очередей

```powershell
# Количество сообщений в очередях
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl list_queues
```

### Просмотр состояния подов

```powershell
# Детальная информация о подах
kubectl describe pod <pod-name>

# Просмотр событий
kubectl get events --sort-by='.lastTimestamp'

# Проверка использования ресурсов
kubectl top nodes
kubectl top pods
```

### Установка Prometheus и Grafana (опционально)

```powershell
# Установка kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack `
  --namespace monitoring `
  --create-namespace

# Port forwarding для Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Откройте: http://localhost:3000
# Username: admin
# Password: prom-operator
```

## Этап 16: Резервное копирование

### Backup PostgreSQL

```bash
# На postgres-db VM
ssh ubuntu@10.0.10.55

# Создание backup
pg_dump -U vidconv -h localhost authdb > /tmp/authdb_backup_$(date +%Y%m%d).sql

# Восстановление
psql -U vidconv -h localhost authdb < /tmp/authdb_backup_20231015.sql
```

### Backup MongoDB

```bash
# На mongo-db VM
ssh ubuntu@10.0.10.56

# Создание backup
mongodump --uri="mongodb://vidconv:StrongPassword123!@localhost:27017/mp3s?authSource=admin" --out=/tmp/mongo_backup_$(date +%Y%m%d)

# Восстановление
mongorestore --uri="mongodb://vidconv:StrongPassword123!@localhost:27017/mp3s?authSource=admin" /tmp/mongo_backup_20231015
```

### Backup MinIO

```bash
# На minio-server VM
ssh ubuntu@10.0.10.54

# Синхронизация бакетов
mc mirror myminio/videos /backup/videos
mc mirror myminio/mp3s /backup/mp3s
```

### Backup Kubernetes манифестов

```powershell
# На Windows машине
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml
```

## Этап 17: Масштабирование

### Горизонтальное масштабирование подов

```powershell
# Увеличение количества реплик
kubectl scale deployment gateway-service --replicas=3
kubectl scale deployment converter-service --replicas=4

# Автоматическое масштабирование (HPA)
kubectl autoscale deployment gateway-service `
  --cpu-percent=70 `
  --min=2 `
  --max=5
```

### Вертикальное масштабирование VM

```hcl
# В terraform/variables.tf измените параметры
"k3s-worker2" = {
  ip     = "10.0.10.53"
  cores  = 6  # было 4
  memory = 12288  # было 8192
  disk   = "50G"
}
```

```powershell
# Применение изменений
terraform apply -auto-approve
```

## Этап 18: Безопасность

### Включение Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gateway-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: gateway-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: auth-service
    ports:
    - protocol: TCP
      port: 5000
  - to:
    - namespaceSelector:
        matchLabels:
          name: rabbitmq
    ports:
    - protocol: TCP
      port: 5672
```

```powershell
kubectl apply -f network-policy.yaml
```

### Настройка SSL/TLS с Let's Encrypt

```powershell
# Установка cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Ожидание готовности
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s
```

```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

```powershell
kubectl apply -f cluster-issuer.yaml
```

### Обновление Ingress для SSL

```yaml
# ingress-ssl.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: video-converter-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - video-converter.yourdomain.com
    secretName: video-converter-tls
  rules:
  - host: video-converter.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway-service
            port:
              number: 8080
```

## Этап 19: CI/CD Pipeline (опционально)

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Build and push auth-service
      uses: docker/build-push-action@v4
      with:
        context: ./auth-service
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/auth-service:latest
    
    - name: Build and push gateway-service
      uses: docker/build-push-action@v4
      with:
        context: ./gateway-service
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/gateway-service:latest
    
    - name: Build and push converter-service
      uses: docker/build-push-action@v4
      with:
        context: ./converter-service
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/converter-service:latest
    
    - name: Build and push notification-service
      uses: docker/build-push-action@v4
      with:
        context: ./notification-service
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/notification-service:latest
    
    - name: Install kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBE_CONFIG }}" > $HOME/.kube/config
    
    - name: Deploy to Kubernetes
      run: |
        kubectl rollout restart deployment auth-service
        kubectl rollout restart deployment gateway-service
        kubectl rollout restart deployment converter-service
        kubectl rollout restart deployment notification-service
```

## Этап 20: Устранение неполадок

### Проблема: Поды не запускаются

```powershell
# Проверка событий
kubectl describe pod <pod-name>

# Проверка логов
kubectl logs <pod-name>

# Проверка образов
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
```

### Проблема: Сервисы недоступны

```powershell
# Проверка endpoints
kubectl get endpoints

# Тест подключения изнутри кластера
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# В контейнере:
curl http://gateway-service:8080/health
```

### Проблема: RabbitMQ не обрабатывает сообщения

```powershell
# Проверка подключения к RabbitMQ
kubectl exec -it <converter-pod> -- sh
# Внутри пода:
python3 -c "import pika; conn = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq.rabbitmq.svc.cluster.local')); print('Connected!')"
```

### Проблема: MinIO недоступен

```bash
# На minio-server VM
ssh ubuntu@10.0.10.54
sudo systemctl status minio

# Проверка логов
sudo journalctl -u minio -f

# Тест подключения
mc admin info myminio
```

### Проблема: Базы данных недоступны

```bash
# PostgreSQL
ssh ubuntu@10.0.10.55
sudo systemctl status postgresql
sudo tail -f /var/log/postgresql/postgresql-14-main.log

# MongoDB
ssh ubuntu@10.0.10.56
sudo systemctl status mongod
sudo tail -f /var/log/mongodb/mongod.log
```

## Заключение

Вы развернули полнофункциональное микросервисное приложение для конвертации видео в аудио на домашней инфраструктуре Proxmox.

### Основные достижения

- ✅ Создана инфраструктура из 6 виртуальных машин
- ✅ Развернут K3s Kubernetes кластер
- ✅ Настроены PostgreSQL и MongoDB
- ✅ Установлен MinIO как S3-совместимое хранилище
- ✅ Развернут RabbitMQ для обработки очередей
- ✅ Настроен Ingress Controller и MetalLB
- ✅ Развернуты 4 микросервиса
- ✅ Настроены email уведомления
- ✅ Организован внешний доступ

### Следующие шаги

1. Настройте регулярные резервные копирования
2. Внедрите мониторинг с Prometheus и Grafana
3. Настройте CI/CD pipeline
4. Добавьте логирование с ELK Stack
5. Улучшите безопасность с Network Policies
6. Задокументируйте процессы для вашего резюме

### Полезные ссылки

- Kubernetes документация: https://kubernetes.io/docs/
- K3s документация: https://docs.k3s.io/
- MinIO документация: https://min.io/docs/
- Terraform Proxmox Provider: https://registry.terraform.io/providers/Telmate/proxmox/latest/docs
- Cloudflare Tunnel: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/

Удачи в вашем проекте! 🚀
