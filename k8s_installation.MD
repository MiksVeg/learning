# Kubernetes HA Setup with Keepalived and HAProxy

This guide describes how to configure a High Availability (HA) Kubernetes cluster using **Keepalived** and **HAProxy**.

---

## /etc/hosts Configuration

```plaintext
192.168.56.109    master-node-1
192.168.56.110    master-node-2
192.168.56.111    worker-node-1
```

---

## /etc/keepalived/chk_haproxy.sh

```bash
#!/bin/bash

# Check if HAProxy systemd service is active
if ! systemctl is-active --quiet haproxy; then
  exit 1
fi

# Check if HAProxy is listening on TCP port 7443
if ! ss -ltn | grep -q ':7443'; then
  exit 1
fi

exit 0
```

Make the script executable:

```bash
chmod +x /etc/keepalived/chk_haproxy.sh
```

---

## /etc/keepalived/keepalived.conf (Master Node 1)

```plaintext
vrrp_script chk_haproxy {
  script "/etc/keepalived/chk_haproxy.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
        state MASTER
        interface enp0s8
        virtual_router_id 1
        priority 100
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass mysecret
        }
        virtual_ipaddress {
                192.168.56.110/24
        }
        track_script {
                chk_haproxy
        }
}
```

---

## /etc/keepalived/keepalived.conf (Master Node 2)

```plaintext
vrrp_script chk_haproxy {
  script "/etc/keepalived/chk_haproxy.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
        state BACKUP
        interface enp0s8
        virtual_router_id 1
        priority 90
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass mysecret
        }
        virtual_ipaddress {
                192.168.56.110/24
        }
        track_script {
                chk_haproxy
        }
}
```

---

## /etc/haproxy/haproxy.cfg

```plaintext
global
    log /dev/log local0
    daemon
    maxconn 4096

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    retries 3

frontend kubernetes-frontend
    bind *:7443
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 3 slowstart 60s maxconn 250 maxqueue 256 weight 100

    server master-node-1 192.168.56.106:6443 check
    server master-node-2 192.168.56.107:6443 check
```

---

## Kubernetes Installation Steps

```bash
sudo swapoff -a
sudo sed -e '/swap/s/^/#/g' -i /etc/fstab
free -m

sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sestatus

sudo systemctl disable --now firewalld
systemctl status firewalld

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

lsmod | grep br_netfilter
lsmod | grep overlay

sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo dnf install containerd.io -y

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo sed -i 's/\(SystemdCgroup =\).*/\1 true/' /etc/containerd/config.toml

sudo cat /etc/containerd/config.toml | grep SystemdCgroup

sudo systemctl enable --now containerd
systemctl status containerd

sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

sudo kubeadm init   --control-plane-endpoint=192.168.56.110:7443   --upload-certs   --apiserver-advertise-address=192.168.56.106   --pod-network-cidr=192.168.0.0/16

helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium --version 1.17.6 --namespace kube-system
```

---

## Notes
- Replace IP addresses as needed for your environment.
- Ensure both **master nodes** can run `haproxy` and `keepalived`.
- Virtual IP (`192.168.56.110`) will be managed by Keepalived.
