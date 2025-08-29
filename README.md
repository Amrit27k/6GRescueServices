# 6G-RESCUE Services Deployment with Ansible

This guide provides step-by-step instructions for deploying the 6G-RESCUE FastAPI Application and IoT(Jetson Orin/Nano) Application using Ansible automation on edge servers or containers.

## Overview

This Ansible playbook automates the deployment of:
- **6G-RESCUE FastAPI Backend** - Python-based API server with ML capabilities
- **Four MEC Services (S1-S4)** - Complete MLOps pipeline for edge AI
- **System dependencies** - Python, virtual environments, networking
- **Application configuration** - Environment files, startup scripts

**Deployment Target**: Ubuntu 20.04+ edge servers or Docker containers

**Note**: Once the services are installed, you can follow to perform further deployment and installation of application by navigating through the `how-to/` directory e2e execution markdown.

## MEC Services Architecture

### S1 - ML Compute Service (JupyterHub)
**Actors**: MEC application developers, data scientists, and AI researchers requiring isolated computational environments for ML model training and experimentation.

**Access & Credentials**: 
- Access via `http://<public-ip>`
- Individual user accounts with dedicated notebook environments
- Pre-configured ML libraries and isolated workspaces
- Login credentials and containerized environments for edge development

### S2 - ML Model Tracking and Remote Deployment Service (MLflow)
**Actors**: ML engineers, DevOps teams, and automated CI/CD pipelines responsible for model lifecycle management and deployment.

**Access & Credentials**:
- Custom MLflow plugin via 6GRescueServices repository
- MLflow tracking URIs and API tokens for model registration
- SSH-based deployment credentials for target edge devices
- Specialized deployment plugins for IoT devices

### S3 - Provisioning and Deployment Service (Ansible)
**Actors**: System administrators, infrastructure engineers, and automated orchestration systems managing edge device provisioning.

**Access & Credentials**:
- Ansible playbooks and Semaphore UI interface
- Ansible inventory access and SSH keys for device management
- Playbook execution credentials for Ubuntu/Debian and IoT systems
- Automated k3s deployment and device configuration

### S4 - ML Model Conversion Service (FogMLaaS)
**Actors**: ML developers and deployment pipelines requiring model optimization for diverse edge hardware platforms.

**Access & Credentials**:
- REST API endpoints at `http://localhost:8000`
- API access tokens and Docker container management credentials
- Model conversion capabilities for edge-specific formats
- Hardware optimization for resource-constrained environments

## Deployment Workflows

### 1. Deploy Frontend & Backend
- Setup React frontend and backend Application for Edge Servers
- Configure frontend-backend communication

### 2. Deploy JupyterHub (S1)
- Set up JupyterHub for ML operations
- Configure user authentication
- Connect to backend services

### 3. Deploy MLflow Services (S2)
- Install MLflow tracking server
- Configure model registry and deployment plugins
- Set up experiment tracking

### 4. Deploy Ansible Services (S3)
- Configure Ansible playbooks for edge deployment
- Set up device inventory and SSH access
- Deploy automation workflows
- Setup React frontend and backend Application for Edge Servers
- Configure frontend-backend communication
- Configuration of Ansible playbook for IoT(Jetson Orin/Nano)
- Deployment of edge applications

### 5. Deploy Model Conversion (S4)
- Set up FogMLaaS Docker container
- Configure API endpoints
- Test model conversion capabilities

## Prerequisites

### Control Machine (where you run Ansible)
- Ubuntu 20.04+ / CentOS 8+ / macOS 10.15+
- Python 3.8+
- Internet connection
- SSH access to target servers

### Target Servers (edge servers)
- Ubuntu 20.04+ (recommended)
- SSH server running
- User with sudo privileges
- Internet connection
- Docker installed (for S4 service)

## Installation & Setup

### 1. Install Ansible

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

#### Using pip (Universal)
```bash
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install ansible
```

### 2. Install Required Collections
```bash
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix
```

### 3. Clone the Services Repository
```bash
git clone https://github.com/Amrit27k/6GRescueServices.git
cd 6GRescueServices
```

## Service Installation Steps

### S1 - ML Compute Service (JupyterHub)

**Installation:**
```bash
# Install The Littlest JupyterHub using the installation URL
curl -L https://tljh.jupyter.org/bootstrap.py | sudo -E python3 - --admin <admin-user-name>

# Alternative: Follow TheLittlestJupyterhub documentation
# https://tljh.jupyter.org/en/latest/install/custom-server.html
```

**Verification:**
- Access JupyterHub at `http://<public-ip>`
- Login with admin credentials
- Verify notebook environment functionality

### S2 - ML Model Tracking Service (MLflow)

**Installation:**
```bash
# Clone the Services Repository (if not already done)
git clone https://github.com/Amrit27k/6GRescueServices.git
cd 6GRescueServices

# Install MLflow plugin
cd Service-S2/mlflow_plugin
pip install -e .

# Confirm installation
python3 -m pip list | grep mlflow
```

**Configuration:**
```bash
# Start MLflow tracking server
mlflow server --host 0.0.0.0 --port 5000 --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns

# Configure MLflow tracking URI
export MLFLOW_TRACKING_URI=http://localhost:5000
```

### S3 - Provisioning Service (Ansible)

**Installation:**
```bash
# Install Ansible (Ubuntu/Debian)
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# Alternative: Using pip
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install ansible

# Install required collections
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix
```

**Configuration:**
```bash
# Run Ansible playbook for full deployment
ansible-playbook -i inventories/inventory.yml playbooks/deploy-full-stack.yml -e @group_vars/all.yml
```

### S4 - ML Model Conversion Service (FogMLaaS)

**Installation:**
```bash
# Clone and build FogMLaaS
git clone https://github.com/tszydlo/FogMLaaS.git
cd FogMLaaS

# Build Docker image
docker build -t fogmlaas:latest .

# Run the container
docker run -d --name fogmlaas -p 8000:8000 fogmlaas:latest
```

**Verification:**
- API available at `http://localhost:8000`
- Test endpoint: `curl http://localhost:8000/docs`

## Container Testing Environment

For safe testing without affecting production servers, use Docker containers:

### 1. Create Test Container
```bash
docker run -d \
  --name rescue-test-target \
  --privileged \
  -p 2222:22 \
  -p 8080:8080 \
  -p 5000:5000 \
  -p 8000:8000 \
  ubuntu:22.04 \
  /bin/bash -c "
    apt-get update && 
    apt-get install -y openssh-server python3 python3-pip python3-venv sudo git curl rsync systemd docker.io && 
    useradd -m -s /bin/bash ubuntu && 
    usermod -aG sudo,docker ubuntu &&
    echo 'ubuntu ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && 
    mkdir -p /var/run/sshd /home/ubuntu/.ssh &&
    chown ubuntu:ubuntu /home/ubuntu/.ssh &&
    chmod 700 /home/ubuntu/.ssh &&
    echo 'ubuntu:password' | chpasswd && 
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && 
    sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config && 
    service docker start &&
    /usr/sbin/sshd -D
  "

# Setup SSH keys
ssh-keygen -t rsa -f ~/.ssh/ansible_test_key -N ""
docker exec rescue-test-target mkdir -p /home/ubuntu/.ssh
docker cp ~/.ssh/ansible_test_key.pub rescue-test-target:/tmp/key.pub
docker exec rescue-test-target bash -c "
    cat /tmp/key.pub >> /home/ubuntu/.ssh/authorized_keys && 
    chown -R ubuntu:ubuntu /home/ubuntu/.ssh &&
    chmod 600 /home/ubuntu/.ssh/authorized_keys &&
    chmod 700 /home/ubuntu/.ssh &&
    rm /tmp/key.pub
"
```

### 2. Test SSH Connection
```bash
ssh -i ~/.ssh/ansible_test_key -p 2222 ubuntu@localhost sudo whoami
```

## Deployment

### 1. Test Connection
```bash
# Test SSH connection to target
ansible all -i inventories/inventory.yml -m ping

# Check sudo access
ansible all -i inventories/inventory.yml -m shell -a "whoami" --become
```

### 2. Syntax Check
```bash
# Check playbook syntax
ansible-playbook -i inventories/inventory.yml playbooks/deploy-mec-services.yml --syntax-check
```

### 3. Dry Run
```bash
# See what would change without making changes
ansible-playbook -i inventories/inventory.yml playbooks/deploy-mec-services.yml -e @group_vars/all.yml --check
```

### 4. Full Deployment
```bash
# Full deployment with verbose output
ansible-playbook -i inventories/inventory.yml playbooks/deploy--services.yml -e @group_vars/all.yml -v
```

## Verification

### 1. Check All Services Status
```bash
# Test all service endpoints
curl http://localhost:8080/docs          # Backend API
curl http://localhost:8080/              # JupyterHub (S1)
curl http://localhost:5000/              # MLflow (S2)
curl http://localhost:8000/docs          # FogMLaaS (S4)

# Check running processes
docker exec rescue-test-target ps aux | grep -E "(uvicorn|jupyter|mlflow)"
```

### 2. Service Health Checks
```bash
# Check service availability
ansible all -i inventories/inventory.yml -m uri -a "url=http://localhost:8080/health"
ansible all -i inventories/inventory.yml -m uri -a "url=http://localhost:5000"
ansible all -i inventories/inventory.yml -m uri -a "url=http://localhost:8000/docs"

# Check system resources
ansible all -i inventories/inventory.yml -m shell -a "df -h"
ansible all -i inventories/inventory.yml -m shell -a "free -m"
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Service Port Conflicts
```bash
# Check port usage
ss -tlnp | grep -E "(8080|5000|8000)"

# Kill conflicting processes
sudo pkill -f uvicorn
sudo pkill -f mlflow
sudo docker stop fogmlaas
```

#### 2. SSH Connection Problems
```bash
# Remove old host keys
ssh-keygen -f ~/.ssh/known_hosts -R '[localhost]:2222'

# Test SSH manually
ssh -i ~/.ssh/ansible_test_key -p 2222 ubuntu@localhost

# Debug connection with maximum verbosity
ansible all -i inventories/inventory.yml -m ping -vvv
```

#### 3. Service Dependencies
```bash
# Check Docker service
sudo systemctl status docker

# Restart services in correct order
sudo systemctl restart docker
docker restart fogmlaas
```

## Success Indicators

A successful deployment will show:
- **S1 JupyterHub**: Accessible at configured port with user authentication
- **S2 MLflow**: Tracking server running with model registry functionality
- **S3 Ansible**: Playbooks executable with proper inventory access
- **S4 FogMLaaS**: REST API responding at `http://localhost:8000/docs`
- **Backend API**: FastAPI server running on port 8080
- **Integration**: All services can communicate and share data
- **Monitoring**: Process monitoring showing all services healthy
- **Logs**: Application logs showing successful startup and operation

## Performance Monitoring

### Resource Usage
```bash
# Monitor CPU and memory usage
ansible all -i inventories/inventory.yml -m shell -a "top -bn1 | head -20"

# Check disk usage
ansible all -i inventories/inventory.yml -m shell -a "du -sh /opt/rescue/*"

# Monitor network connections
ansible all -i inventories/inventory.yml -m shell -a "ss -tlnp"
```

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review service-specific logs in `/var/log/rescue/`
3. Test individual service components
4. Consult the 6G-RESCUE project documentation
5. Verify network connectivity between services
6. Check resource utilization and scaling requirements

---

**Project**: 6G-RESCUE Services   
**Last Updated**: August 2025  
**Version**: 1.0.0  