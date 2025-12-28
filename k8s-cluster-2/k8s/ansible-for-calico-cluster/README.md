# Kubernetes Ansible Deployment

This Ansible configuration automates the deployment and setup of Kubernetes clusters with Calico networking, Metal-LB load balancing, and multi-cluster context management.

## Overview

This Ansible playbook setup is designed to be executed from the **master node** of each Kubernetes cluster. The playbook automates:

- **Kubernetes Master Node Setup**: Installation and configuration of the Kubernetes control plane
- **Kubernetes Worker Node Setup**: Installation and configuration of worker nodes
- **Network Configuration**: Deployment of Calico CNI for pod networking
- **Load Balancing**: Configuration of Metal-LB for service exposure
- **Multi-Cluster Context**: Creation of kubectl contexts for managing multiple clusters

## Project Structure

```
ansible-for-calico-cluster/
├── site.yml                          # Main playbook (entry point)
├── hosts                             # Inventory file with node definitions
├── k8s-contexts-template.conf        # Template for kubeconfig context configuration
│
├── kubernetes_master/                # Role for master node configuration
│   ├── tasks/main.yml               # Master node setup tasks
│   ├── handlers/main.yml            # Handlers for service restarts
│   ├── defaults/main.yml            # Default variables
│   ├── vars/main.yml                # Role-specific variables
│   ├── templates/                   # Jinja2 templates
│   ├── files/                       # Static files
│   ├── tests/                       # Test configuration
│   └── README.md                    # Role documentation
│
├── kubernetes_worker/                # Role for worker node configuration
│   ├── tasks/main.yml               # Worker node setup tasks
│   ├── handlers/main.yml            # Handlers for service restarts
│   ├── defaults/main.yml            # Default variables
│   ├── vars/main.yml                # Role-specific variables
│   ├── templates/                   # Jinja2 templates
│   ├── files/                       # Static files
│   ├── tests/                       # Test configuration
│   └── README.md                    # Role documentation
│
├── kubernetes_network/               # Role for Calico networking setup
│   └── tasks/main.yml               # Network configuration tasks
│
├── kubernetes_metal_lb/              # Role for Metal-LB setup
│   └── tasks/main.yml               # Metal-LB configuration tasks
│
└── kubernetes_create_contexts/       # Role for multi-cluster context setup
    └── tasks/main.yml               # Context creation tasks
```

## Requirements

### System Requirements
- **OS**: Ubuntu 20.04 LTS or later (other Debian-based distributions may work)
- **Python 3**: Python 3.8 or later on all target nodes
- **Ansible**: Version 2.9 or later on the control machine
- **Network Access**: SSH access from the control machine to all cluster nodes

### On Each Cluster Master Node
- Ubuntu 20.04 LTS or later pre-installed
- SSH server running and accessible
- Sudo privileges for the Ansible user (or root access)
- At least 2 CPU cores and 2GB RAM
- Internet access for downloading packages

### On Each Cluster Worker Node
- Ubuntu 20.04 LTS or later pre-installed
- SSH server running and accessible
- Sudo privileges for the Ansible user (or root access)
- At least 2 CPU cores and 2GB RAM
- Network connectivity to the master node(s)

## Execution Model

### Important: Local Execution on Master Node

The Kubernetes installation **must be executed from the master node itself**. This is critical because the cluster bootstrap process requires local system access.

**Root path for execution**: `/root/k8s/ansible-for-calico-cluster/`

**Do NOT use**: `$pwd/K8S-DEPLOY...` or relative paths from your development machine.

### Proper Execution Flow

1. Copy the Ansible playbook to each cluster's master node at `/root/k8s/`
2. SSH into the master node
3. Navigate to `/root/k8s/ansible-for-calico-cluster/`
4. Execute the playbook locally on the master node

```bash
# On the master node
cd /root/k8s/ansible-for-calico-cluster/
ansible-playbook -i hosts site.yml
```

## Configuration

### Inventory File (hosts)

The `hosts` file defines the cluster topology. Edit this file to match your environment:

```ini
[master_nodes]
cluster-2-master-1 ansible_host=10.200.1.101

[worker_nodes]
cluster-2-worker-1 ansible_host=10.200.1.102
cluster-2-worker-2 ansible_host=10.200.1.103

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

**Key Configuration Items:**
- `master_nodes`: Hostnames/IPs of Kubernetes master nodes
- `worker_nodes`: Hostnames/IPs of Kubernetes worker nodes
- `ansible_host`: IP address or hostname for SSH connection
- `ansible_python_interpreter`: Path to Python 3 on target nodes (must be `/usr/bin/python3`)

### kubeconfig Context Template

The `k8s-contexts-template.conf` file provides a template for creating kubectl contexts that work across multiple clusters. This is used by the `kubernetes_create_contexts` role to generate proper kubeconfig entries.

## Usage

### Prerequisites on Control Machine

1. Install Ansible:
```bash
pip install ansible>=2.9
```

2. Generate SSH keys (if not already done):
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
```

3. Configure SSH access to cluster nodes (passwordless recommended):
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub user@master_node_ip
ssh-copy-id -i ~/.ssh/id_rsa.pub user@worker_node_ip
```

### Step 1: Prepare Master Node

Copy the Ansible playbook to each cluster's master node:

```bash
# From your development machine
scp -r ansible-for-calico-cluster/ root@cluster-1-master:/root/k8s/

# For cluster 2
scp -r ansible-for-calico-cluster/ root@cluster-2-master:/root/k8s/
```

### Step 2: Execute on Master Node

SSH into each master node and execute the playbook:

```bash
# SSH into cluster 1 master
ssh root@cluster-1-master

# Navigate to the playbook directory
cd /root/k8s/ansible-for-calico-cluster/

# Update the hosts file for your environment
vi hosts

# Execute the playbook
ansible-playbook -i hosts site.yml
```

**Playbook Execution Flow:**
1. Configures Kubernetes Master node
2. Sets up network (Calico CNI)
3. Configures Worker nodes
4. Creates multi-cluster contexts
5. Configures Metal-LB for load balancing

### Step 3: Verify Installation

After the playbook completes successfully:

```bash
# Check cluster nodes
kubectl get nodes

# Verify all pods are running
kubectl get pods -A

# Check Calico network status
kubectl get pods -n kube-system -l k8s-app=calico-node

# Check Metal-LB
kubectl get pods -n metallb-system
```

## Roles Explained

### kubernetes_master
Installs and configures the Kubernetes control plane:
- Docker and container runtime
- Kubernetes API server, controller manager, scheduler
- etcd database
- Required kernel modules and system settings
- kubelet configuration

### kubernetes_worker
Prepares worker nodes to join the cluster:
- Docker and container runtime
- kubelet configuration
- Kernel modules and system settings
- Join token configuration (when integrated with master role)

### kubernetes_network
Deploys Calico Container Network Interface:
- Calico operator installation
- BGP configuration for pod networking
- Network policy setup

### kubernetes_metal_lb
Configures Metal-LB for load balancing:
- Metal-LB controller installation
- IP pool configuration
- BGP peer configuration

### kubernetes_create_contexts
Creates and manages kubectl contexts:
- Generates kubeconfig entries for multiple clusters
- Allows seamless switching between clusters
- Configures context-specific credentials

## Tags

The playbook uses Ansible tags for granular execution control:

- `pre_requisites`: Install and configure prerequisites (kernel modules, system settings)
- Other role-specific tags available in individual role documentation

### Run Specific Tags

```bash
# Only install prerequisites
ansible-playbook -i hosts site.yml --tags=pre_requisites

# Skip specific tags
ansible-playbook -i hosts site.yml --skip-tags=pre_requisites
```

## Troubleshooting

### Common Issues

#### 1. Python Interpreter Not Found
**Error**: `Failed to set permissions on the temporary directory`

**Solution**: Ensure `ansible_python_interpreter=/usr/bin/python3` is set in the hosts file.

```bash
# On target nodes, verify Python location
which python3
```

#### 2. SSH Connection Refused
**Error**: `FAILED - UNREACHABLE!`

**Solution**: 
```bash
# Verify SSH connectivity
ssh -i your_key user@master_node_ip

# Check SSH service
sudo systemctl status ssh
```

#### 3. Permission Denied
**Error**: `Failed to set permissions on temporary file`

**Solution**: Ensure the Ansible user has sudo privileges:
```bash
# On target nodes
sudo usermod -aG sudo ansible_user
```

#### 4. Insufficient Resources
**Error**: Pod failures due to memory/CPU

**Solution**: Increase node resources or adjust resource requests in the cluster configuration.

### Debug Mode

Run the playbook in verbose mode for detailed troubleshooting:

```bash
# Verbose output
ansible-playbook -i hosts site.yml -v

# Very verbose (print all variable values)
ansible-playbook -i hosts site.yml -vv

# Extra verbose (include system module arguments)
ansible-playbook -i hosts site.yml -vvv
```

## Multi-Cluster Deployment

For deploying to multiple clusters:

### Cluster 1
```bash
# Copy to cluster 1 master
scp -r ansible-for-calico-cluster/ root@cluster-1-master:/root/k8s/

# SSH and execute
ssh root@cluster-1-master
cd /root/k8s/ansible-for-calico-cluster/
# Update hosts file for cluster 1
vi hosts
ansible-playbook -i hosts site.yml
```

### Cluster 2
```bash
# Copy to cluster 2 master
scp -r ansible-for-calico-cluster/ root@cluster-2-master:/root/k8s/

# SSH and execute
ssh root@cluster-2-master
cd /root/k8s/ansible-for-calico-cluster/
# Update hosts file for cluster 2
vi hosts
ansible-playbook -i hosts site.yml
```

## Security Considerations

1. **SSH Key Security**: Protect your private SSH keys
   ```bash
   chmod 600 ~/.ssh/id_rsa
   ```

2. **Become Method**: The playbook uses `become: yes` to execute tasks with elevated privileges. Ensure proper sudoers configuration.

3. **Network Security**: Restrict SSH access to authorized machines only

4. **Kubeconfig Protection**: Protect kubeconfig files with restricted permissions
   ```bash
   chmod 600 ~/.kube/config
   ```

5. **Vault for Secrets**: For production, consider using Ansible Vault:
   ```bash
   ansible-playbook -i hosts site.yml --ask-vault-pass
   ```

## Maintenance and Updates

### Update Kubernetes Version

Modify relevant tasks in `kubernetes_master/tasks/main.yml` and `kubernetes_worker/tasks/main.yml` to target the desired Kubernetes version.

### Cluster Health Checks

```bash
# Check node status
kubectl get nodes -o wide

# Check system pods
kubectl get pods -A

# Check persistent volumes
kubectl get pv

# Check services
kubectl get svc -A
```

### Cleanup (If Needed)

To remove the Kubernetes cluster:
```bash
# Reset worker nodes
ansible-playbook reset.yml -i hosts

# Drain and remove master
kubectl drain <node-name> --ignore-daemonsets
kubectl delete node <node-name>
```

## Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Calico Documentation](https://docs.projectcalico.org/)
- [Metal-LB Documentation](https://metallb.universe.tf/)

## Support and Contributions

For issues or contributions:
1. Review existing documentation and troubleshooting sections
2. Check system logs: `journalctl -u kubelet`
3. Verify kubeconfig: `kubectl config view`
4. Check Ansible logs: Add `-vv` or `-vvv` flags for more detail

## License

This Ansible configuration is part of the Kubernetes multi-cluster deployment project.

## Author Information

For internal use only. Maintained by the ASE team.
