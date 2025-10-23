# -*- mode: ruby -*-
# vi: set ft=ruby :

# Kubernetes 1.34.1 cluster with containerd and Calico CNI
# 1 control-plane node + 2 worker nodes

VAGRANTFILE_API_VERSION = "2"

# Cluster configuration
KUBERNETES_VERSION = "1.34.1"
POD_NETWORK_CIDR = "10.244.0.0/16"
CONTROL_PLANE_IP = "192.168.57.10"

# Node definitions
NODES = [
  { name: "k8s-control-plane", ip: "192.168.57.10", cpus: 2, memory: 6144, role: "control-plane" },
  { name: "k8s-worker-1", ip: "192.168.57.11", cpus: 2, memory: 2048, role: "worker" },
  { name: "k8s-worker-2", ip: "192.168.57.12", cpus: 2, memory: 2048, role: "worker" }
]

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.box_check_update = false

  # Increase boot timeout
  config.vm.boot_timeout = 600

  # SSH configuration
  config.ssh.insert_key = true
  config.ssh.forward_agent = true

  # Iterate over each node
  NODES.each_with_index do |node, index|
    config.vm.define node[:name] do |node_config|
      node_config.vm.hostname = node[:name]
      node_config.vm.network "private_network", ip: node[:ip]

      # Provider configuration - Parallels for ARM64 Macs
      node_config.vm.provider "parallels" do |prl|
        prl.name = node[:name]
        prl.memory = node[:memory]
        prl.cpus = node[:cpus]
        prl.update_guest_tools = false
      end

      # BINARIES ONLY MODE - NO CONFIGURATION APPLIED
      # Installs: containerd, kubeadm, kubelet, kubectl
      # Does NOT configure: swap, sysctl, kernel modules, /etc/hosts, services
      # Run ALL configuration manually per MANUAL_STEPS.md
      node_config.vm.provision "ansible" do |ansible|
        ansible.playbook = "playbooks/binaries-only.yml"
      end
    end
  end
end
