# 6G-RESCUE Backend Deployment with Ansible

This guide provides step-by-step instructions for deploying the 6G-RESCUE FastAPI backend using Ansible automation on edge servers or containers.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Installation & Setup](#installation--setup)
- [Container Testing Environment](#container-testing-environment)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

## Overview

This Ansible playbook automates the deployment of:
- **6G-RESCUE FastAPI Backend** - Python-based API server with ML capabilities
- **System dependencies** - Python, virtual environments, networking
- **Application configuration** - Environment files, startup scripts
- **Service management** - Background process management

**Deployment Target**: Ubuntu 20.04+ edge servers or Docker containers

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

## Project Structure

```
~/ansible-rescue-deployment/
├── inventories/
│   └── inventory.yml              # Server inventory
├── playbooks/
│   └── deploy-backend.yml         # Main deployment playbook
├── roles/
│   ├── system-prep/
│   │   ├── tasks/main.yml         # System preparation tasks
│   │   └── handlers/main.yml      # System event handlers
│   └── rescue-backend/
│       ├── tasks/main.yml         # Backend deployment tasks
│       ├── templates/
│       │   ├── backend.env.j2     # Environment configuration
│       │   ├── start-backend.sh.j2 # Startup script
│       │   └── rescue-backend.service.j2 # Systemd service (optional)
│       └── handlers/main.yml      # Backend event handlers
├── group_vars/
│   └── all.yml                    # Global configuration variables
└── ansible.cfg                    # Ansible configuration
```

## Installation & Setup

### 1. Install Ansible

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

#### CentOS/RHEL/Fedora
```bash
sudo dnf install -y epel-release
sudo dnf install -y ansible
```

#### macOS
```bash
brew install ansible
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

### 3. Create Project Directory
```bash
mkdir -p ~/ansible-rescue-deployment
cd ~/ansible-rescue-deployment

# Create directory structure
mkdir -p inventories playbooks group_vars
mkdir -p roles/{system-prep,rescue-backend}/{tasks,templates,handlers}
```

## Container Testing Environment

For safe testing without affecting production servers, use Docker containers:

### 1. Create Test Container Script
```bash
cat > create-test-container.sh << 'EOF'
#!/bin/bash

echo "🧹 Cleaning up existing container..."
docker stop rescue-test-target 2>/dev/null || true
docker rm rescue-test-target 2>/dev/null || true

echo "🐳 Creating new test container..."
docker run -d \
  --name rescue-test-target \
  --privileged \
  -p 2222:22 \
  -p 8080:8080 \
  ubuntu:22.04 \
  /bin/bash -c "
    apt-get update && 
    apt-get install -y openssh-server python3 python3-pip python3-venv sudo git curl rsync systemd && 
    useradd -m -s /bin/bash ubuntu && 
    usermod -aG sudo ubuntu &&
    echo 'ubuntu ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && 
    mkdir -p /var/run/sshd /home/ubuntu/.ssh &&
    chown ubuntu:ubuntu /home/ubuntu/.ssh &&
    chmod 700 /home/ubuntu/.ssh &&
    echo 'ubuntu:password' | chpasswd && 
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && 
    sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config && 
    /usr/sbin/sshd -D
  "

echo "⏳ Waiting for container to be ready..."
sleep 15

echo "🔑 Setting up SSH keys..."
if [ ! -f ~/.ssh/ansible_test_key ]; then
    ssh-keygen -t rsa -f ~/.ssh/ansible_test_key -N ""
fi

docker cp ~/.ssh/ansible_test_key.pub rescue-test-target:/tmp/key.pub
docker exec rescue-test-target bash -c "
    cat /tmp/key.pub >> /home/ubuntu/.ssh/authorized_keys && 
    chown ubuntu:ubuntu /home/ubuntu/.ssh/authorized_keys &&
    chmod 600 /home/ubuntu/.ssh/authorized_keys &&
    rm /tmp/key.pub
"

echo "✅ Testing SSH connection..."
if ssh -i ~/.ssh/ansible_test_key -p 2222 -o StrictHostKeyChecking=no ubuntu@localhost whoami > /dev/null 2>&1; then
    echo "🎉 Container setup successful!"
    echo "🚀 You can now run the deployment!"
else
    echo "❌ SSH connection failed. Check container logs:"
    docker logs rescue-test-target
fi
EOF

chmod +x create-test-container.sh
./create-test-container.sh
```

## Configuration

### 1. Ansible Configuration (`ansible.cfg`)
```ini
[defaults]
host_key_checking = False
inventory = inventories/inventory.yml
remote_tmp = /tmp/.ansible-${USER}
stdout_callback = yaml
callback_whitelist = timer, profile_tasks

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining = True
```

### 2. Inventory Configuration (`inventories/inventory.yml`)

#### For Container Testing:
```yaml
all:
  hosts:
    edge-server:
      ansible_host: localhost
      ansible_port: 2222
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/ansible_test_key
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

#### For Real Servers:
```yaml
all:
  hosts:
    edge-server:
      ansible_host: 10.70.0.64  # Your server IP
      ansible_user: ubuntu      # Your SSH user
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

### 3. Global Variables (`group_vars/all.yml`)
```yaml
---
# 6G-RESCUE Application Configuration
rescue_app:
  git_repo: "https://github.com/Amrit27k/6GRescueApplication.git"
  branch: "main"
  build_directory: "/tmp/rescue-build"

# Backend Configuration (FastAPI)
backend:
  host: "0.0.0.0"              # For containers, use 0.0.0.0
  port: 8080
  directory: "/opt/rescue-backend"
  python_version: "3.10"

# Backend Environment Variables
backend_env:
  jupyterhub_url: "http://localhost"
  jupyterhub_user: "akumar"
  jupyterhub_token: "test_token"
  jetson_ip: "192.168.2.100"
  mqtt_broker: "127.0.0.1"
  mqtt_port: 1883

# System Configuration
system_packages:
  - python3
  - python3-pip
  - python3-venv
  - nodejs
  - npm
  - git
  - curl
  - wget
  - htop
  - ufw
  - rsync
```

### 4. Main Playbook (`playbooks/deploy-backend.yml`)
```yaml
---
- name: Deploy 6G-RESCUE Backend Only
  hosts: all
  become: yes
  gather_facts: yes
  
  tasks:
    - name: Print deployment info
      debug:
        msg: |
          Deploying 6G-RESCUE Backend to: {{ ansible_host }}
          Backend will run on: http://{{ backend.host }}:{{ backend.port }}
  
  roles:
    - system-prep
    - rescue-backend
    
  post_tasks:
    - name: Wait for backend to be ready
      uri:
        url: "http://localhost:8080/"
        method: GET
        status_code: 200
      retries: 10
      delay: 2
      delegate_to: localhost
      become: no
      
    - name: Display success message
      debug:
        msg: |
          🎉 Backend deployment complete!
          
          🌐 Access your backend:
          - API Documentation: http://localhost:8080/docs
          - API Base URL: http://localhost:8080/
          
          🔍 Check status:
          - Service status: docker exec rescue-test-target ps aux | grep uvicorn
          - View logs: docker exec rescue-test-target cat /var/log/rescue/backend.log
```

### 5. Create Role Files

Copy the complete role files from the previous guides:
- `roles/system-prep/tasks/main.yml` - System preparation
- `roles/rescue-backend/tasks/main.yml` - Backend deployment
- `roles/rescue-backend/templates/` - Configuration templates
- `roles/rescue-backend/handlers/main.yml` - Event handlers

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
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml --syntax-check
```

### 3. Dry Run
```bash
# See what would change without making changes
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml --check
```

### 4. Deploy
```bash
# Full deployment
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml

# Deploy with verbose output
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml -v
```

## Verification

### 1. Check Deployment Status
```bash
# Test API endpoints
curl http://localhost:8080/docs
curl http://localhost:8080/

# Check process status
docker exec rescue-test-target ps aux | grep uvicorn

# View application logs
docker exec rescue-test-target cat /var/log/rescue/backend.log
```

### 2. API Testing
```bash
# View interactive documentation
open http://localhost:8080/docs  # or visit in browser

# Check API schema
curl http://localhost:8080/openapi.json

# Test specific endpoints (depends on your FastAPI app)
curl http://localhost:8080/health
```

### 3. Service Management
```bash
# View running processes
ansible all -i inventories/inventory.yml -m shell -a "ps aux | grep uvicorn"

# Check system resources
ansible all -i inventories/inventory.yml -m shell -a "htop -n 1"

# Check network connections
ansible all -i inventories/inventory.yml -m shell -a "ss -tlnp | grep 8080"
```

## Troubleshooting

### Common Issues and Solutions

#### 1. SSH Connection Problems
```bash
# Remove old host keys
ssh-keygen -f ~/.ssh/known_hosts -R '[localhost]:2222'

# Test SSH manually
ssh -i ~/.ssh/ansible_test_key -p 2222 ubuntu@localhost

# Debug connection
ansible all -i inventories/inventory.yml -m ping -vvv
```

#### 2. Permission Issues
```bash
# Check user permissions
ansible all -i inventories/inventory.yml -m shell -a "groups $USER"

# Check sudo access
ansible all -i inventories/inventory.yml -m shell -a "sudo whoami"
```

#### 3. Backend Not Starting
```bash
# Check logs
docker exec rescue-test-target cat /var/log/rescue/backend.log

# Check virtual environment
docker exec rescue-test-target ls -la /opt/rescue-backend/venv/bin/

# Test manual startup
docker exec -it rescue-test-target bash
cd /opt/rescue-backend
source venv/bin/activate
python -c "import fastapi; print('FastAPI imported successfully')"
```

#### 4. Port Issues
```bash
# Check port availability
docker exec rescue-test-target ss -tlnp | grep 8080

# Test port forwarding
telnet localhost 8080
```

### Debugging Commands
```bash
# Run with maximum verbosity
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml -vvv

# Start at specific task
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml --start-at-task "Create Python virtual environment"

# Run only specific tags
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml --tags backend

# Limit to specific hosts
ansible-playbook -i inventories/inventory.yml playbooks/deploy-backend.yml -e @group_vars/all.yml --limit edge-server
```

## Next Steps

After successful backend deployment:

### 1. Deploy Frontend
- Create React frontend role
- Configure frontend-backend communication
- Set up Nginx reverse proxy

### 2. Deploy JupyterHub
- Set up JupyterHub for ML operations
- Configure user authentication
- Connect to backend services

### 3. Production Deployment
- Configure SSL certificates
- Set up monitoring and logging
- Implement backup strategies
- Configure load balancing

### 4. CI/CD Integration
- Set up automated testing
- Configure deployment pipelines
- Implement rolling updates

## Success Indicators

A successful deployment will show:
- ✅ **FastAPI server** running on port 8080
- ✅ **Swagger UI** accessible at `http://localhost:8080/docs`
- ✅ **API responses** returning JSON data
- ✅ **Process monitoring** showing uvicorn running
- ✅ **Log files** showing successful startup

## Support

For issues or questions:
1. Check the troubleshooting section
2. Review application logs
3. Test individual components
4. Consult the 6G-RESCUE project documentation

---

**Project**: 6G-RESCUE Edge ML Operations System   
**Last Updated**: August 2025  
**Version**: 1.0.0
