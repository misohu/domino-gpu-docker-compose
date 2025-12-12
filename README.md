# Domino + Airflow Docker Compose Setup (GPU-enabled)

This repository contains a production-ready Docker Compose setup for running Domino with Apache Airflow, including GPU support for ML workloads.

## Architecture

- **Domino Services**: REST API, Frontend, PostgreSQL database
- **Airflow Services**: API Server, Scheduler, DAG Processor, Celery Worker (GPU-enabled), Triggerer
- **Supporting Services**: Redis (Celery broker), Docker proxy
- **GPU Support**: Celery worker has access to all available NVIDIA GPUs

## Prerequisites

### 1. Install Docker & Docker Compose

#### Ubuntu/Debian
```bash
# Update package index
sudo apt-get update

# Install required packages
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to docker group (logout/login required)
sudo usermod -aG docker $USER
```

**Official Documentation**: https://docs.docker.com/engine/install/

### 2. Install NVIDIA Drivers

Ensure you have NVIDIA GPU drivers installed on your host system.

```bash
# Check if drivers are installed
nvidia-smi

# If not installed, install the latest drivers
sudo apt-get update
sudo apt-get install -y nvidia-driver-535  # or latest version
sudo reboot
```

**Official Documentation**: https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/

### 3. Install NVIDIA Container Toolkit

The NVIDIA Container Toolkit allows Docker containers to access GPU resources.

```bash
# Configure the repository
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install the NVIDIA Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Test GPU access in Docker
docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

**Official Documentation**: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html

## Configuration

### Environment Variables

Create a `.env` file in the root directory (optional - defaults are provided):

```bash
# Domino Configuration
DOMINO_GIT_BRANCH=main
DOMINO_DEPLOY_MODE=local-compose
DOMINO_FRONTEND_BASENAME=/
DOMINO_DEFAULT_PIECES_REPOSITORY_VERSION=main
LOCAL_DOMINO_SHARED_DATA_PATH=./shared_storage

# Airflow Configuration
AIRFLOW_UID=50000
_AIRFLOW_WWW_USER_USERNAME=admin
_AIRFLOW_WWW_USER_PASSWORD=airflow

# GitHub Tokens (if needed)
DOMINO_DEFAULT_PIECES_REPOSITORY_TOKEN=
DOMINO_GITHUB_ACCESS_TOKEN_WORKFLOWS=
DOMINO_GITHUB_WORKFLOWS_REPOSITORY=
```

### File Permissions

Set proper ownership for Airflow directories:

```bash
# Set AIRFLOW_UID if needed (default: 50000)
export AIRFLOW_UID=$(id -u)  # or use 50000

# Ensure directories exist
mkdir -p ./dags ./logs ./plugins ./config ./shared_storage

# Set permissions
sudo chown -R ${AIRFLOW_UID}:0 ./dags ./logs ./plugins ./config
sudo chmod -R 755 ./dags ./logs ./plugins ./config
```

## Running the Stack

### Start All Services

```bash
# Build and start all services
docker compose up -d

# View logs
docker compose logs -f

# Check service status
docker compose ps
```

### First-Time Setup

The `airflow-init` service will automatically:
- Initialize the Airflow database
- Create the admin user (default: admin/airflow)
- Set up required directories

This runs automatically on first startup.

## Accessing Services

- **Domino Frontend**: http://localhost:3000
- **Domino REST API**: http://localhost:8000
- **Airflow Web UI**: http://localhost:8080/airflow

### Default Credentials

**Airflow**:
- Username: `admin` (or value from `_AIRFLOW_WWW_USER_USERNAME`)
- Password: `airflow` (or value from `_AIRFLOW_WWW_USER_PASSWORD`)

**Domino**:
- Configured via `CREATE_DEFAULT_USER=true` in domino-rest service

## Managing the Stack

```bash
# Stop all services
docker compose down

# Stop and remove volumes (WARNING: deletes all data)
docker compose down -v

# Restart a specific service
docker compose restart airflow-scheduler

# View logs for a specific service
docker compose logs -f domino-rest

# Execute commands in a container
docker compose exec airflow-apiserver airflow dags list

# Rebuild after code changes
docker compose up -d --build
```

## GPU Verification

Verify GPU access in the Airflow worker:

```bash
# Check GPU in worker container
docker compose exec airflow-domino-worker nvidia-smi
```

## Troubleshooting

### GPU Not Available

1. Verify host GPU access: `nvidia-smi`
2. Check Docker GPU access: `docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi`
3. Ensure NVIDIA Container Toolkit is properly configured
4. Restart Docker daemon: `sudo systemctl restart docker`

### Permission Errors

```bash
# Reset permissions
export AIRFLOW_UID=50000
sudo chown -R ${AIRFLOW_UID}:0 ./dags ./logs ./plugins ./config
```

### Service Won't Start

```bash
# Check logs
docker compose logs [service-name]

# Remove volumes and restart fresh (WARNING: deletes data)
docker compose down -v
docker compose up -d
```

### Database Connection Issues

Ensure PostgreSQL services are healthy:
```bash
docker compose ps | grep postgres
docker compose logs domino-postgres
docker compose logs airflow-postgres
```

## Directory Structure

```
.
├── docker-compose.yaml       # Main compose configuration
├── Dockerfile-airflow-domino # Custom Airflow image with Domino integration
├── dags/                     # Airflow DAGs directory
├── logs/                     # Airflow logs (auto-generated)
├── plugins/                  # Airflow plugins
├── config/                   # Airflow configuration files
│   └── airflow.cfg
└── shared_storage/           # Shared data between services
```

## Networks

- `domino-network`: Communication between Domino services
- `airflow-network`: Communication between Airflow services
- Services requiring cross-communication are connected to both networks

## Volumes

- `domino-postgres-volume`: Persistent Domino database storage
- `airflow-postgres-volume`: Persistent Airflow database storage

## Additional Resources

- **Domino GitHub**: https://github.com/IISAS/domino
- **Apache Airflow Documentation**: https://airflow.apache.org/docs/
- **Docker Documentation**: https://docs.docker.com/
- **NVIDIA Container Toolkit**: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/
- **Docker Compose GPU Support**: https://docs.docker.com/compose/gpu-support/

## License

Refer to the respective Domino and Apache Airflow licenses.
