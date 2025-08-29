# Step-by-Step Deployment

### Step 1: Install Ansible and Dependencies

On your control machine (development server):

#### Ubuntu/Debian Installation
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

#### Universal Installation (using pip)
```bash
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install ansible
```

#### Install Required Ansible Collections
```bash
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix
```

### Step 2: Deploy Edge Infrastructure

Clone and deploy the complete MEC services stack:

```bash
# Clone the services repository
git clone https://github.com/Amrit27k/6GRescueServices.git
cd 6GRescueServices/

# Deploy full infrastructure stack
ansible-playbook -i inventories/inventory.yml playbooks/deploy-full-stack.yml -e @group_vars/all.yml
```

**What this deploys:**
- JupyterHub for ML development (S1)
- MLflow tracking server (S2)  
- Ansible automation services (S3)
- FogMLaaS model conversion (S4)
- Backend API services
- Network configuration

### Step 2.1: Configure MQTT Broker on Edge Server

Set up the MQTT broker for communication between edge server and Jetson devices:

#### Install Mosquitto MQTT Broker
```bash
sudo apt-get update
sudo apt-get install mosquitto mosquitto-clients
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
```

#### Configure Firewall
```bash
# Allow MQTT port (1883) through firewall
sudo ufw allow 1883/tcp

# Verify MQTT broker is running
sudo systemctl status mosquitto
```

#### Install Python MQTT Libraries
```bash
# Install MQTT client library
pip install paho-mqtt

# Verify installation
python3 -c "import paho.mqtt.client as mqtt; print('MQTT library installed successfully')"
```

**Note**: This MQTT broker setup will be integrated into the Ansible playbook in future versions for automated deployment.

### Step 3: Configure ML Training Environment

#### Option A: Manual Edge Setup

```bash
# Clone the face recognition repository
git clone https://github.com/Amrit27k/6GRescue-FaceRecognition.git
cd 6GRescue-FaceRecognition

# Navigate to Jetson-specific code
cd jetson

# Create and activate virtual environment (optional but recommended)
python3 -m venv rescue-env
source rescue-env/bin/activate

# Install dependencies
pip install -r requirements.txt
```

#### Option B: Automated Edge Setup (Recommended)

```bash
# Use the automated Ansible setup from 6GRescueServices
# This automatically configures the edge environment with JupyterHub integration
# Follow the setup instructions in the 6GRescueServices repository documentation

# Access JupyterHub after deployment by navigating to: http://<public-ip>
```

### Step 4: Set Up MLflow Deployment Plugin

Install the MLflow plugin for edge device deployment:

```bash
cd 6GRescue-Services/Services-S2/mlflow_plugins
pip install -e .
```

**Configure SSH Access to Jetson Device:**

```bash
# Generate SSH key for Jetson access
ssh-keygen -t rsa -f ~/.ssh/jetson_key

# Copy public key to Jetson (replace with your Jetson IP)
ssh-copy-id -i ~/.ssh/jetson_key.pub newcastleuni@192.168.2.100

# Test SSH connection
ssh -i ~/.ssh/jetson_key newcastleuni@192.168.2.100 "echo 'SSH connection successful'"
```

**Deploy Model to Jetson:**

```bash
cd mlflow_plugin_examples
python simple_file_transfer.py --iot_ip 192.168.2.100 --model rf
```

### Step 5: Build Docker Images on Jetson Device

SSH into your Jetson device and build the required containers:

```bash
# SSH into Jetson
ssh -i ~/.ssh/jetson_key newcastleuni@192.168.2.100

# Navigate to deployed model files
cd mlflow_deployments_v<version-number>/face_recognition_files

# Build model server container
docker build -f docker/Dockerfile.model-server \
    -t face-model-mlf-plugin:latest .

# Build inference server with RTSP support
docker build -f docker/Dockerfile.inference-server-rtsp \
    -t face-inference-mlf-plugin-rtsp:latest .
```

### Step 6: Deploy and Run Containers

Start the containerized services on the Jetson device:

#### Start Model Server
```bash
docker run -d \
    --name face-model-mlf-plugin \
    --network face-net-mlf-plugin \
    -p 5000:5000 \
    -v $(pwd)/models:/app/models \
    -v $(pwd)/logs:/app/logs \
    --memory=512m \
    --restart unless-stopped \
    face-model-mlf-plugin:latest
```

#### Start Inference Server with RTSP Support
```bash
docker run -d \
    --name face-inference-mlf-plugin-rtsp \
    --runtime=nvidia \
    -p 5001:5001 \
    -p 8554:8554 \
    -v $(pwd)/output:/app/output \
    -v $(pwd)/temp_frames:/app/temp_frames \
    --volume /tmp/argus_socket:/tmp/argus_socket \
    --volume /usr/src/tensorrt/data:/usr/src/tensorrt/data:ro \
    --device /dev/nvhost-ctrl \
    --device /dev/nvhost-ctrl-gpu \
    --device /dev/nvhost-gpu \
    --device /dev/nvhost-as-gpu \
    --device /dev/nvhost-vic \
    --device /dev/nvhost-msenc \
    --device /dev/nvmap \
    -v /dev:/dev \
    --device /dev/video0 \
    --privileged \
    -e MODEL_SERVICE_URL=http://localhost:5000 \
    -e RTSP_AUTO_START=true \
    face-inference-mlf-plugin-rtsp:latest
```

### Step 7: View Live RTSP Stream

From your development machine (outside JupyterHub), view the real-time face detection stream:

```bash
# Run the MQTT client to view RTSP stream with face detection overlays
python3 client-mqtt.py --iot_ip 192.168.2.100 --edge_ip 127.0.0.1
```

## Verification and Testing

### Service Health Checks

```bash
# Check infrastructure services
curl http://<edge-server>:8080/docs    # Backend API
curl http://<edge-server>:5000         # MLflow
curl http://<edge-server>:8000/docs    # FogMLaaS

# Check Jetson services
curl http://192.168.2.100:5000/health  # Model server
curl http://192.168.2.100:5001/status  # Inference server
```

### Container Status
```bash
# On Jetson device
docker ps
docker logs face-model-mlf-plugin
docker logs face-inference-mlf-plugin-rtsp
```

### RTSP Stream Verification
```bash
# Test RTSP stream accessibility
ffprobe rtsp://192.168.2.100:8554/stream

# Or use VLC player
vlc rtsp://192.168.2.100:8554/stream
```