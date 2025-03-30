# Raspberry Pi Kubernetes Cluster

This documentation serves as a comprehensive guide for individuals looking to learn about Kubernetes clustering on Raspberry Pi devices as a hobby project, putting essential setup and configuration information in one place.

## Hardware Configuration
- **Master**: Raspberry Pi 4 8GB (named PI4-Cluster)
- **Nodes**: 4 Raspberry Pi Zero 2W (named P1, P2, P3, P4)
- **Important note**: The nodes are in a NAT and are only accessible from the master

## Preparation and Installation

### SD Card Configuration
- [ ] Master: PI4-CLUSTER
- [ ] Node 1: P1
- [ ] Node 2: P2
- [ ] Node 3: P3
- [ ] Node 4: P4

### ClusterCTRL Commands

```bash
# Power control
clusterctrl on                # Turn on all Pi Zeros
clusterctrl off               # Turn off all Pi Zeros
clusterctrl on p1             # Turn on only the Pi Zero in slot P1
clusterctrl on p1 p3 p4       # Turn on Pi Zeros in slots P1, P3 and P4
clusterctrl off p2 p3         # Turn off Pi Zeros in slots P2 and P3

# LED control
clusterctrl alert on          # Turn on the alert LED
clusterctrl alert off         # Turn off the alert LED
clusterctrl led on            # Enable Power & P1-P4 LEDs (default)
clusterctrl led off           # Disable Power & P1-P4 LEDs

# Other commands
clusterctrl hub on            # Enable USB hub (default)
clusterctrl hub off           # Disable USB hub
clusterctrl wp on             # Write-protect the HAT EEPROM
clusterctrl wp off            # Disable write protection (needed only for updates)


# Status check
clusterctrl status            # Display cluster status
lsusb -t                      # List USB devices
ip addr show                  # Display network configurations
```

### SSH Configuration on PI4-Cluster

```bash
# SSH key generation
ssh-keygen -t ed25519

# Copy SSH keys to nodes, Replace your USER by your USERNAME
ssh-copy-id -i ~/.ssh/id_ed25519.pub USER@p1.local
ssh-copy-id -i ~/.ssh/id_ed25519.pub USER@p2.local
ssh-copy-id -i ~/.ssh/id_ed25519.pub USER@p3.local
ssh-copy-id -i ~/.ssh/id_ed25519.pub USER@p4.local
```

- [ ] Node 1: SSH key copy
- [ ] Node 2: SSH key copy
- [ ] Node 3: SSH key copy
- [ ] Node 4: SSH key copy

## Kubernetes Installation

### Kubectl Installation

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl.sha256"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Raspberry Pi Configuration for K3S

```bash
# Edit config.txt
sudo nano /boot/firmware/config.txt
# Add or uncomment:
# dtoverlay=disable-bt
# dtoverlay=disable-wifi
# gpu_mem=16

# Edit cmdline.txt
sudo nano /boot/firmware/cmdline.txt
# Add to the end of the line:
# cgroup_memory=1 cgroup_enable=memory
```

### K3S Installation

#### On the Master

```bash
sudo su -
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -s -
sudo cat /var/lib/rancher/k3s/server/node-token
```

#### On the Nodes (Pi Zero 2W)

```bash
# Replace token and URL with your actual values
curl -sfL https://get.k3s.io | K3S_TOKEN="SERVER_TOKEN" K3S_URL="https://pi4-cluster.local:6443" K3S_NODE_NAME=$(hostname) sh -

# Verification
systemctl status k3s-agent.service
journalctl -u k3s-agent.service -f
```

### K3S Installation Verification

```bash
# Check service
systemctl status k3s

# Check nodes
kubectl get nodes
```

## Rancher Installation

### Helm Installation

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Cert-manager Installation

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.6/cert-manager.yaml
```

### Rancher Installation

```bash
# Add Rancher repo
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

# Create namespace
kubectl create namespace cattle-system

# Install Rancher
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=pi4-cluster.local \
  --set bootstrapPassword=admin \
  --set replicas=1 \
  --set auditLog.level=1

# Check installation
kubectl -n cattle-system rollout status deploy/rancher

# Get setup URL with password
echo https://pi4-cluster.local/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

# Get bootstrap password
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```
Once installation finished, you should be able to access your Ranch Web Interface and setup your Admin password

### Important Rancher Parameters

- **replicas=1**: Configuration with a single instance (suitable for Raspberry Pi)
- **hostname**: External URL to access the Rancher interface
- **bootstrapPassword**: Initial administrator password
- **auditLog.level=1**: Basic audit logging level
