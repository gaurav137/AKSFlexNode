# AKS Flex Node

A Go service that extends Azure Kubernetes Service (AKS) to non-Azure VMs through Azure Arc integration, enabling hybrid and edge computing scenarios.

**Status:** Work In Progress
**Platform:** Ubuntu 22.04.5 LTS (tested)
**Architecture:** x86_64 (amd64)

## Overview

AKS Flex Node transforms any Ubuntu VM into a fully managed AKS worker node by:

- 🔗 **Azure Arc Registration** - Registers your VM with Azure Arc for cloud management
- 📦 **Container Runtime Setup** - Installs and configures runc and containerd
- ☸️ **Kubernetes Integration** - Deploys kubelet, kubectl, and kubeadm components
- 🌐 **Network Configuration** - Sets up Container Network Interface (CNI) for pod networking
- 🚀 **Service Orchestration** - Configures and manages all required systemd services
- ⚡ **Cluster Connection** - Securely joins your VM as a worker node to your existing AKS cluster

## Data Flow

```mermaid
sequenceDiagram
    participant E as 💻 Edge VM
    participant A as 🤖 aks-flex-node
    participant Arc as 🔗 Azure Arc
    participant K as ☸️ AKS Cluster

    Note over E,K: 1. Initial Setup Phase
    E->>A: Start agent
    A->>Arc: Register with Arc (HTTPS)
    Arc-->>A: Managed identity created

    Note over E,K: 2. Bootstrap Phase
    A->>A: Install container runtime
    A->>A: Install Kubernetes components
    A->>A: Configure system settings

    Note over E,K: 3. Cluster Join Phase
    A->>K: Request cluster credentials (HTTPS)
    K-->>A: Provide kubeconfig
    A->>E: Configure kubelet service
    E->>K: Join as worker node (HTTPS/443)

    Note over E,K: 4. Runtime Operations
    K->>E: Schedule workloads
    E->>K: Report node status
    E->>K: Stream logs & metrics
```

## Quick Start

### Prerequisites
- **VM Requirements:**
  - Ubuntu 22.04.5 LTS VM (non-Azure)
  - Minimum 2GB RAM, 25GB free disk space
  - Sudo access on the VM
- **AKS Cluster Requirements:**
  - Azure RBAC enabled AKS cluster
  - Network connectivity from edge VM to cluster API server (port 443)
- **Azure Authentication & Permissions:**
  - The user account (when using `az login`) or service principal (when configured in config file) needs permissions to register the edge node with Azure Arc and to assign the Arc managed identity the necessary RBAC permissions for AKS cluster access

### 1. Authenticate with Azure
```bash
# Login with an account that has necessary permissions
az login
```

### 2. Build and Install
```bash
# Clone the repository
git clone <repository-url>
cd AKSFlexNode

# Build the binary
go build -o aks-flex-node .

# Install system-wide
sudo cp aks-flex-node /usr/local/bin/
```

### 3. Configure
Create the configuration directory and file:

```bash
# Create configuration directory
sudo mkdir -p /etc/aks-flex-node

# Create configuration file
sudo tee /etc/aks-flex-node/config.json > /dev/null << 'EOF'
{
  "azure": {
    "subscriptionId": "your-subscription-id",
    "tenantId": "your-tenant-id",
    "cloud": "AzurePublicCloud",
    "arc": {
      "machineName": "your-unique-node-name",
      "tags": {
        "environment": "edge",
        "node-type": "worker"
      },
      "resourceGroup": "your-resource-group",
      "location": "westus",
      "autoRoleAssignment": true
    },
    "targetCluster": {
      "resourceId": "/subscriptions/your-subscription-id/resourceGroups/your-rg/providers/Microsoft.ContainerService/managedClusters/your-cluster",
      "location": "westus"
    }
  },
  "agent": {
    "logLevel": "info",
    "logDir": "/var/log/aks-flex-node"
  }
}
EOF

```

**Important:** Replace the placeholder values with your actual Azure resource information:
- `your-subscription-id`: Your Azure subscription ID
- `your-tenant-id`: Your Azure tenant ID
- `your-unique-node-name`: A unique name for this edge node
- `your-resource-group`: Resource group where Arc machine and AKS cluster are located
- `your-cluster`: Your AKS cluster name

### 5. Bootstrap the Node
```bash
# Transform your VM into an AKS node
aks-flex-node bootstrap --config /etc/aks-flex-node/config.json
```

### 6. Verify Installation
```bash
# Check if the node has joined the cluster
kubectl get nodes

# Verify Arc registration
az connectedmachine list --resource-group your-resource-group
```

## Usage Modes

### 🛠️ Development Mode
**Best for:** Testing, development, and one-off deployments
**Authentication:** Uses Azure CLI credentials from the user who installed the service

```bash
# Bootstrap with explicit config path (uses Azure CLI credentials)
aks-flex-node bootstrap --config /etc/aks-flex-node/config.json

# Clean removal of all components
aks-flex-node unbootstrap --config /etc/aks-flex-node/config.json
```

**Note:** The service automatically gains access to Azure CLI credentials during installation. Make sure you've run `az login` before installing the service.

### 🏭 Production Mode
**Best for:** Automated deployments and production environments
**Authentication:** Service principal credentials in config file

Add service principal credentials to your config file:
```json
{
  "azure": {
    "subscriptionId": "your-subscription-id",
    "tenantId": "your-tenant-id",
    "servicePrincipal": {
      "clientId": "your-service-principal-client-id",
      "clientSecret": "your-service-principal-client-secret"
    },
    "cloud": "AzurePublicCloud",
    // ... rest of config
  }
}
```

```bash
# Install and enable systemd services
sudo ./AKSFlexNode/install-service.sh

sudo systemctl enable aks-flex-node@bootstrap.service
sudo systemctl start aks-flex-node@bootstrap.service

# Monitor bootstrap progress
sudo systemctl status aks-flex-node@bootstrap.service
sudo journalctl -u aks-flex-node@bootstrap -f

# Uninstall (and unbootstrap node)
sudo ./AKSFlexNode/uninstall-service.sh
```

## Available Commands

| Command | Description | Usage |
|---------|-------------|-------|
| `bootstrap` | Transform VM into AKS node | `sudo aks-flex-node bootstrap` |
| `unbootstrap` | Clean removal of all components | `sudo aks-flex-node unbootstrap` |
| `version` | Show version information | `sudo aks-flex-node version` |

## System Requirements

- **Operating System:** Ubuntu 22.04.5 LTS
- **Architecture:** x86_64 (amd64)
- **Memory:** Minimum 2GB RAM (4GB recommended)
- **Storage:**
  - **Minimum:** 25GB free space
  - **Recommended:** 40GB free space
  - **Production:** 50GB+ free space
- **Network:** Internet connectivity to Azure endpoints
- **Privileges:** Root/sudo access required
- **Build Dependencies:** Go 1.23+ (if building from source)

### Storage Breakdown
- **Base components:** ~3GB (Arc agent, runc, containerd, Kubernetes binaries, CNI plugins)
- **System directories:** ~5-10GB (`/var/lib/containerd`, `/var/lib/kubelet`, configurations)
- **Container images:** ~5-15GB (pause container, system images, workload images)
- **Logs:** ~2-5GB (`/var/log/pods`, `/var/log/containers`, agent logs)
- **Installation buffer:** ~5-10GB (temporary downloads, garbage collection headroom)


## Documentation [TO BE ADDED]

- [Development Guide](docs/DEVELOPMENT.md)
- [Configuration Reference](docs/CONFIGURATION.md)
- [Setup Guide](docs/AKS_EDGE_NODE_SETUP_GUIDE.md)
- [Troubleshooting](docs/TROUBLESHOOTING.md)
- [APT Packaging](docs/APT_PACKAGING_GUIDE.md)

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

- Report issues: [GitHub Issues](https://github.com/your-org/AKSFlexNode/issues)
- Discussion: [GitHub Discussions](https://github.com/your-org/AKSFlexNode/discussions)
- Email: support@yourorg.com

---

<div align="center">

**🚀 Built with ❤️ for the Kubernetes community**

![Made with Go](https://img.shields.io/badge/Made%20with-Go-00ADD8?style=flat-square&logo=go)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Ready-326CE5?style=flat-square&logo=kubernetes)
![Azure](https://img.shields.io/badge/Azure-Integrated-0078D4?style=flat-square&logo=microsoftazure)

</div>