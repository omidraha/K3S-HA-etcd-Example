# HA K3s Cluster with Embedded etcd

This guide walks you through deploying a high-availability K3s cluster with an embedded etcd datastore on three nodes,
using a Vagrant environment.
It also shows how to disable the default add-ons (traefik, servicelb, local‑storage) that K3s deploys by default.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Vagrant Environment Setup](#vagrant-environment-setup)
- [Pre-Installation Steps](#pre-installation-steps)
- [Cluster Installation](#cluster-installation)
  - [Node 1 (Master Node)](#node-1-master-node)
  - [Node 2 and Node 3 (Worker Nodes)](#node-2-and-node-3-worker-nodes)
- [Verification and Troubleshooting](#verification-and-troubleshooting)
- [Clean Up](#clean-up)
- [References and Useful Links](#references-and-useful-links)

---

## Prerequisites

- **Vagrant** and **VirtualBox** must be installed on your machine.
- An internet connection is required to download the K3s binaries and images.
- SSH keys properly set up for Vagrant provisioning (used in the Vagrantfile).
- Sufficient system resources for each node (e.g. at least 2GB RAM per node).

---

## Vagrant Environment Setup

Create a file named `Vagrantfile` with the following content.
This Vagrantfile defines three nodes with specific IP addresses, memory, and disk settings:

```ruby
Vagrant.configure("2") do |config|
  IMAGE = "ubuntu/jammy64"

  NODES = [
    { name: "node1", ip: "192.168.56.11", ram: 2048, disk: "15GB" },
    { name: "node2", ip: "192.168.56.12", ram: 2048, disk: "15GB" },
    { name: "node3", ip: "192.168.56.13", ram: 2048, disk: "15GB" }
  ]

  NODES.each do |node|
    config.vm.define node[:name] do |node_config|
      node_config.vm.box = IMAGE
      node_config.vm.hostname = node[:name]
      node_config.vm.network "private_network", ip: node[:ip]

      # Set disk size (requires the vagrant-disksize plugin)
      node_config.disksize.size = node[:disk]

      node_config.vm.provider "virtualbox" do |vb|
        vb.memory = node[:ram]
        vb.cpus = 1
      end

      # Provisioning: Set password, grant sudo privileges, and add SSH key
      node_config.vm.provision "shell", inline: <<-SHELL
        #!/bin/bash
        echo "ubuntu:ubuntu" | chpasswd
        usermod -aG sudo ubuntu
        mkdir -p /home/ubuntu/.ssh
        echo "#{File.read('~/.ssh/id_rsa.pub')}" >> /home/ubuntu/.ssh/authorized_keys
        chown -R ubuntu:ubuntu /home/ubuntu/.ssh
        chmod 700 /home/ubuntu/.ssh
        chmod 600 /home/ubuntu/.ssh/authorized_keys
      SHELL
    end
  end
end
```

Bring up your Vagrant environment:

```bash
vagrant up
```

---

## Pre-Installation Steps

Before installing K3s on each node, execute the following steps on every node to ensure the system is ready:

1. **Load Required Kernel Modules**

   ```bash
   sudo modprobe br_netfilter
   sudo modprobe ip_tables
   sudo modprobe xt_mark
   sudo modprobe overlay
   sudo modprobe nf_conntrack
   ```

   Verify `xt_mark` is loaded:

   ```bash
   lsmod | grep xt_mark
   ```

2. **Disable IPv6 (Optional)**

   To disable IPv6 temporarily:

   ```bash
   sudo sysctl net.ipv6.conf.all.disable_ipv6=1
   sudo sysctl net.ipv6.conf.default.disable_ipv6=1
   sudo sysctl -p
   ```

3. **Configure iptables to Use Legacy Mode**

   ```bash
   sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
   sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
   ```

---

## Cluster Installation

### Node 1 (Master Node)

On **Node 1**, initialize the K3s cluster with embedded etcd
and disable the default add-ons (traefik, servicelb, local-storage). Run:

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=SuperSecretValue sh -s - server \
  --cluster-init \
  --tls-san 192.168.56.11 \
  --advertise-address=192.168.56.11 \
  --node-ip=192.168.56.11 \
  --etcd-expose-metrics \
  --disable=traefik,servicelb,local-storage
```

After the cluster is up, retrieve the **node-token** from Node 1:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

This token which will look similar to this, is used to join the other nodes to the cluster.
```
K10xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx::server:SuperSecretValue
```

---

### Node 2 and Node 3 (Worker Nodes)

On **Node 2**, run the following command (replace `TOKEN_FROM_NODE1` with the node token you retrieved):

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --server https://192.168.56.11:6443 \
  --token TOKEN_FROM_NODE1 \
  --node-ip=192.168.56.12 \
  --advertise-address=192.168.56.12 \
  --tls-san 192.168.56.11 \
  --disable=traefik,servicelb,local-storage" sh -
```

On **Node 3**, run a similar command (update the node name and IP accordingly):

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --server https://192.168.56.11:6443 \
  --token TOKEN_FROM_NODE1 \
  --node-ip=192.168.56.13 \
  --advertise-address=192.168.56.13 \
  --tls-san 192.168.56.11 \
  --disable=traefik,servicelb,local-storage" sh -
```

---

## Verification and Troubleshooting

### Verify the Cluster

Check that all nodes are registered and in the ready state:

```bash
sudo kubectl get nodes -o wide
```

Expected output similar to:

```
NAME    STATUS   ROLES                       AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node1   Ready    control-plane,etcd,master   3m48s   v1.32.3+k3s1   192.168.56.11   <none>        Ubuntu 24.04.2 LTS   6.8.0-57-generic   containerd://2.0.4-k3s2
node2   Ready    control-plane,etcd,master   61s     v1.32.3+k3s1   192.168.56.12   <none>        Ubuntu 24.04.2 LTS   6.8.0-57-generic   containerd://2.0.4-k3s2
node3   Ready    control-plane,etcd,master   4s      v1.32.3+k3s1   192.168.56.13   <none>        Ubuntu 24.04.2 LTS   6.8.0-57-generic   containerd://2.0.4-k3s2
```

Also, check that the essential pods are running:

```bash
sudo kubectl get pods -A -o wide
```

Expected output similar to:

```
sudo kubectl get pods -A -o wide
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE   IP          NODE    NOMINATED NODE   READINESS GATES
kube-system   coredns-ff8999cc5-6x2cv           1/1     Running   0          32m   10.42.0.2   node1   <none>           <none>
kube-system   metrics-server-6f4c6675d5-tfdv9   1/1     Running   0          32m   10.42.0.3   node1   <none>           <none>
```

### Troubleshooting Common Issues

- **APIService Metrics Errors:**
  Errors regarding "failed to download v1beta1.metrics.k8s.io" are common if the metrics server is slow or temporarily unreachable. Monitor the pod logs:

  ```bash
  sudo kubectl -n kube-system logs -l k8s-app=metrics-server
  ```

- **Sync Failures with iptables-restore:**
  If you see errors such as
  ```
  ip6tables-restore v1.8.10 (legacy): unknown option "--xor-mark"
  ```
  this is due to using the legacy iptables version, and these messages are usually non‑fatal. Confirm the system is using legacy iptables mode.

- **Node NotReady or Network Issues:**
  Verify your nodes have the correct IP addresses configured:

  ```bash
  ip a | grep inet
  ```

  Also, ensure that ports used by etcd (e.g. 2379) and the API server (6443) are open:

  ```bash
  sudo netstat -tulpn | grep 2379
  sudo netstat -tulpn | grep 6443
  ```

---

## Clean Up

If you need to remove K3s from a node, run:

```bash
sudo /usr/local/bin/k3s-uninstall.sh
sudo rm -rf /etc/rancher/k3s /var/lib/rancher/k3s /var/lib/kubelet /var/lib/cni /run/k3s
```

---

## References and Useful Links

- **K3s HA with Embedded etcd Documentation:**
  [https://docs.k3s.io/datastore/ha-embedded](https://docs.k3s.io/datastore/ha-embedded)

- **Related K3s Issue:**
  [https://github.com/k3s-io/k3s/issues/11175](https://github.com/k3s-io/k3s/issues/11175)

- **K3s GitHub Repository:**
  [https://github.com/k3s-io/k3s](https://github.com/k3s-io/k3s)

- **Vagrant Documentation:**
  [https://www.vagrantup.com/docs](https://www.vagrantup.com/docs)

