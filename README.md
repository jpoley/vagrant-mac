# Kubernetes 1.34.1 Cluster on Vagrant

A local 3-node Kubernetes cluster (1 control plane + 2 workers) provisioned with Vagrant and Ansible on Parallels for ARM64 Macs.

## ⚠️ CURRENT STATUS: FULLY AUTOMATED PROVISIONING

This project now includes **complete automated provisioning**. Running `vagrant up` will:
- ✅ Install and configure containerd runtime
- ✅ Install Kubernetes binaries (kubelet, kubeadm, kubectl)
- ✅ Initialize the control plane with kubeadm
- ✅ Install Calico CNI
- ✅ Join worker nodes to the cluster
- ✅ Untaint control plane for workload scheduling

The cluster will be **fully functional** after provisioning completes.

## Prerequisites

- **Parallels Desktop** (required for ARM64 Macs)
- **Vagrant** 2.3.0 or later
- **Ansible** installed on host machine
- At least **11 GB RAM** available (cluster uses 10 GB total)
- At least **30 GB disk space**

## Quick Start

```bash
# Clone and enter the directory
cd /path/to/vagrant-mac

# Start the entire cluster (takes 10-15 minutes first time)
vagrant up

# Verify cluster is running
./verify-cluster.sh

# SSH into the control plane
vagrant ssh k8s-control-plane

# Check cluster status from control plane
kubectl get nodes
kubectl get pods -A
```

## Cluster Configuration

| Component | Details |
|-----------|---------|
| **Kubernetes Version** | 1.34.1 |
| **Container Runtime** | containerd 1.7.28 |
| **CNI Plugin** | Calico 3.28.0 |
| **Base OS** | Ubuntu 24.04 LTS |
| **Control Plane IP** | 192.168.57.10 |
| **Worker 1 IP** | 192.168.57.11 |
| **Worker 2 IP** | 192.168.57.12 |
| **Pod Network CIDR** | 10.244.0.0/16 |

### Node Resources

- **Control Plane**: 2 CPU, 6144 MB RAM
- **Worker 1**: 2 CPU, 2048 MB RAM
- **Worker 2**: 2 CPU, 2048 MB RAM

## Common Commands

### Cluster Management

```bash
# Start all nodes
vagrant up

# Start specific node
vagrant up k8s-control-plane
vagrant up k8s-worker-1
vagrant up k8s-worker-2

# Stop the cluster
vagrant halt

# Restart with re-provisioning
vagrant reload --provision

# Destroy the cluster
vagrant destroy -f

# Check cluster status
vagrant status
```

### SSH Access

```bash
# SSH into nodes
vagrant ssh k8s-control-plane
vagrant ssh k8s-worker-1
vagrant ssh k8s-worker-2

# Run single command without interactive shell
vagrant ssh k8s-control-plane -c "kubectl get nodes"
```

### Kubernetes Operations

```bash
# From control plane node
vagrant ssh k8s-control-plane

# Inside control plane:
kubectl get nodes
kubectl get pods -A
kubectl get namespaces
kubectl cluster-info

# Deploy a test workload
kubectl create deployment nginx --image=nginx
kubectl get pods
```

## Project Structure

```
.
├── Vagrantfile                      # VM definitions and provisioning order
├── README.md                        # This file
├── CLAUDE.md                        # Developer/AI assistant guidance
├── verify-cluster.sh                # Health check script
└── playbooks/
    ├── binaries-only.yml            # Install k8s binaries
    ├── common.yml                   # System configuration
    ├── containerd.yml               # Container runtime setup
    ├── control-plane.yml            # Control plane initialization
    ├── calico.yml                   # Calico CNI installation (active)
    ├── cilium.yml                   # Cilium CNI installation (alternative)
    ├── untaint.yml                  # Allow control plane scheduling
    ├── workers.yml                  # Worker node join
    └── k8s-join-command.sh          # Join command (auto-generated)
```

## Troubleshooting

### Workers Won't Join

If workers fail to join with token errors:

```bash
# Generate fresh join command
vagrant ssh k8s-control-plane -c "sudo kubeadm token create --print-join-command"

# Copy output to playbooks/k8s-join-command.sh

# Re-provision workers
vagrant provision k8s-worker-1 --provision-with ansible
vagrant provision k8s-worker-2 --provision-with ansible
```

### Cluster Not Responding

```bash
# Check if VMs are running
vagrant status

# Check kubelet status on control plane
vagrant ssh k8s-control-plane -c "sudo systemctl status kubelet"

# Check containerd status
vagrant ssh k8s-control-plane -c "sudo systemctl status containerd"

# View kubelet logs
vagrant ssh k8s-control-plane -c "sudo journalctl -u kubelet -f"
```

### Pods Not Ready

```bash
# Check pod status
vagrant ssh k8s-control-plane -c "kubectl get pods -A"

# Check Calico CNI status
vagrant ssh k8s-control-plane -c "kubectl get pods -n kube-system -l k8s-app=calico-node"

# View pod logs
vagrant ssh k8s-control-plane -c "kubectl logs -n kube-system <pod-name>"
```

### Start Fresh

If the cluster is completely broken:

```bash
# Destroy and rebuild from scratch
vagrant destroy -f
vagrant up
```

## Modifying the Cluster

### Change Kubernetes Version

1. Edit `Vagrantfile` and update `KUBERNETES_VERSION` variable
2. Rebuild: `vagrant destroy -f && vagrant up`

### Change Node Resources

1. Edit the `NODES` array in `Vagrantfile`
2. Modify `cpus` or `memory` values
3. Apply changes: `vagrant reload`

### Add More Workers

1. Add new node to `NODES` array in `Vagrantfile`
2. Add node to `/etc/hosts` section in `playbooks/common.yml`
3. Start the new node: `vagrant up <new-node-name>`

## Accessing the Cluster from Host

To use `kubectl` from your host machine:

```bash
# Copy kubeconfig from control plane
vagrant ssh k8s-control-plane -c "cat ~/.kube/config" > ~/.kube/vagrant-k8s-config

# Use the config
export KUBECONFIG=~/.kube/vagrant-k8s-config
kubectl get nodes
```

Note: You may need to update the server address in the config from `127.0.0.1:6443` to `192.168.57.10:6443`.

## Additional Documentation

- **CLAUDE.md** - Detailed technical documentation, architecture decisions, and AI assistant guidance
- **k8s-broken.md** - Root cause analysis of common cluster failures (useful for learning)

## Support

For issues specific to this setup, check:
1. Vagrant logs: `vagrant up` output
2. Ansible provisioning output
3. Cluster verification: `./verify-cluster.sh`
4. Kubelet logs: `vagrant ssh <node> -c "sudo journalctl -u kubelet"`

## License

This is a development/learning environment. Use at your own risk.
