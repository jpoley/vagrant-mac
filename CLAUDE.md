# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Vagrant-based Kubernetes 1.34.1 cluster automation project using Ansible. It provisions a local 3-node cluster (1 control-plane + 2 workers) with containerd as the container runtime and Calico CNI. **FULLY AUTOMATED** - a single `vagrant up` command creates a production-ready cluster.

## Cluster Configuration

- **Kubernetes Version**: 1.34.1 (defined in Vagrantfile)
- **CNI**: Calico v3.28.0
- **Pod Network CIDR**: 10.244.0.0/16
- **Node Names**: k8s-cp (control plane), k8s-node-1, k8s-node-2
- **Node IPs**: 192.168.57.10 (control plane), 192.168.57.11, 192.168.57.12
- **Container Runtime**: containerd.io v1.7.28
- **Base Image**: bento/ubuntu-24.04 (for Parallels on ARM64)
- **Hypervisor**: Parallels (required for ARM64 Macs)

## Current Status

**FULLY OPERATIONAL** - Complete automated provisioning with `vagrant up`:
- ✅ System configuration: swap disabled, kernel modules loaded, sysctl configured, /etc/hosts updated
- ✅ Container runtime: containerd configured with systemd cgroup driver and running
- ✅ Kubernetes binaries: kubeadm, kubelet, kubectl v1.34.1 installed and running
- ✅ Control plane: initialized with kubeadm, kubeconfig configured for vagrant and root users
- ✅ CNI: Calico v3.28.0 installed and operational
- ✅ Worker nodes: joined to cluster and Ready
- ✅ Control plane scheduling: untainted for workload scheduling

The cluster is production-ready immediately after `vagrant up` completes.

## Common Commands

### Cluster Management
```bash
# Start the entire cluster (all nodes)
vagrant up

# Start a specific node
vagrant up k8s-cp
vagrant up k8s-node-1
vagrant up k8s-node-2

# Halt/stop the cluster
vagrant halt

# Destroy the cluster
vagrant destroy -f

# SSH into nodes
vagrant ssh k8s-cp
vagrant ssh k8s-node-1
vagrant ssh k8s-node-2

# Reload with re-provisioning
vagrant reload --provision
```

### Cluster Verification
```bash
# Run the verification script
./verify-cluster.sh

# Manual verification from control plane
vagrant ssh k8s-cp -c "kubectl get nodes -o wide"
vagrant ssh k8s-cp -c "kubectl get pods -A"

# Check Calico status
vagrant ssh k8s-cp -c "kubectl get pods -n kube-system -l k8s-app=calico-node"
```

### Re-provisioning
```bash
# Re-run specific provisioning steps
vagrant provision k8s-cp --provision-with ansible

# Force re-provisioning of all nodes
vagrant destroy -f && vagrant up
```

## Architecture

### Provisioning Flow

The automated provisioning follows this sequence:

1. **common.yml** - System configuration for all nodes
   - Disable swap and configure kernel modules (overlay, br_netfilter)
   - Set up networking sysctls for Kubernetes
   - Add all cluster nodes to /etc/hosts

2. **binaries-only.yml** - Kubernetes binary installation
   - Adds Docker repository and GPG key for containerd
   - Installs containerd.io package (v1.7.28)
   - Adds Kubernetes repository and GPG key
   - Installs kubelet, kubeadm, kubectl at version 1.34.1
   - Holds Kubernetes packages to prevent auto-updates

3. **containerd.yml** - Container runtime configuration
   - Configure CRI plugin and systemd cgroup driver
   - Critical: Set `SystemdCgroup = true` in config.toml
   - Enable and start containerd service

4. **control-plane.yml** - Control plane initialization (control plane only)
   - Run `kubeadm init` to initialize the cluster
   - Generate join command and save to `playbooks/k8s-join-command.sh`
   - Set up kubeconfig for both vagrant and root users

5. **calico.yml** - CNI installation (control plane only)
   - Download Calico v3.28.0 manifest
   - Deploy Calico CNI
   - Wait for Calico pods to become ready

6. **untaint.yml** - Allow control plane scheduling (control plane only)
   - Remove NoSchedule taint from control plane

7. **workers.yml** - Worker node join (worker nodes only)
   - Copy join command from playbooks directory
   - Join worker to the cluster using kubeadm join

### Key Design Decisions

- **Parallels-only**: VirtualBox doesn't support ARM64 properly on Apple Silicon Macs
- **Fully automated**: Single `vagrant up` command provisions complete, ready-to-use cluster
- **Calico CNI**: Using Calico v3.28.0 for pod networking (kube-proxy included)
- **Control plane scheduling**: Control plane is untainted to allow workload scheduling (suitable for dev/test)
- **Short hostnames**: k8s-cp, k8s-node-1, k8s-node-2 for easier CLI usage

### Important Files

- **Vagrantfile** - Node definitions, resource allocation, provisioning orchestration
- **playbooks/k8s-join-command.sh** - Auto-generated during control-plane provisioning, contains kubeadm join token (gitignored for security)
- **verify-cluster.sh** - Health check script for the cluster
- **.gitignore** - Excludes k8s-join-command.sh (contains sensitive tokens)

## Modifying the Cluster

### Changing Kubernetes Version
1. Update `KUBERNETES_VERSION` in Vagrantfile (e.g., "1.35.0")
2. Ensure the version exists in the Kubernetes apt repository
3. Run `vagrant destroy -f && vagrant up`

### Changing Node Resources
1. Edit the `NODES` array in Vagrantfile
2. Modify `cpus` or `memory` values
3. Run `vagrant reload` to apply changes

### Changing Network Configuration
1. Update `POD_NETWORK_CIDR` or node IPs in Vagrantfile
2. Update corresponding IPs in `playbooks/common.yml` (lines 67-69)
3. Full reprovisioning required: `vagrant destroy -f && vagrant up`

### Adding Additional Workers
1. Add new node definition to `NODES` array in Vagrantfile
2. Add corresponding entry in `playbooks/common.yml` /etc/hosts section
3. Run `vagrant up <new-node-name>`

## Troubleshooting

### Join Command Issues
If workers fail to join:
1. Check `playbooks/k8s-join-command.sh` exists and contains valid token
2. Token expires after 24 hours - regenerate on control-plane:
   ```bash
   vagrant ssh k8s-cp
   kubeadm token create --print-join-command
   ```
3. Update `playbooks/k8s-join-command.sh` with new command
4. Reprovision workers: `vagrant provision k8s-node-1 --provision-with ansible`

### Calico Not Ready
1. Check Calico pods: `vagrant ssh k8s-cp -c "kubectl get pods -n kube-system -l k8s-app=calico-node"`
2. View Calico controller: `vagrant ssh k8s-cp -c "kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers"`
3. View logs: `vagrant ssh k8s-cp -c "kubectl logs -n kube-system -l k8s-app=calico-node"`

### Node NotReady
1. Verify containerd is running: `vagrant ssh <node> -c "systemctl status containerd"`
2. Check kubelet logs: `vagrant ssh <node> -c "journalctl -u kubelet -f"`
3. Verify CNI is operational on control-plane before expecting workers to be Ready
