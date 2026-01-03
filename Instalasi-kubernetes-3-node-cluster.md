# 📘 Kubernetes 3-Node Cluster Setup Guide

### Kubernetes v1.35.0 + containerd + Calico Tigera

Dokumentasi ini menjelaskan langkah-demi-langkah membuat cluster Kubernetes dengan:

- 1 Control Plane (Master)
- 2 Worker Node
- Container runtime: **containerd**
- CNI: **Calico Tigera**
- Pod CIDR: **10.244.0.0/16**

---

## 🧩 1. Arsitektur & Topologi

Cluster akan memiliki struktur seperti di bawah ini:

| Role         | Hostname      | IP Address      | OS               | CPU | RAM |
|-------------|---------------|----------------|------------------|-----|-----|
| Master Node | `k8s-master`  | `192.168.2.104` | Debian 13 Trixie | 2+  | 4GB |
| Worker 1    | `k8s-worker1` | `192.168.2.105` | Debian 13 Trixie | 2+  | 4GB |
| Worker 2    | `k8s-worker2` | `192.168.2.106` | Debian 13 Trixie | 2+  | 4GB |

**Minimal Spesifikasi Rekomendasi**
- CPU: 2 core
- RAM: 4GB
- Disk: 20GB+
- Internet aktif

> Pastikan semua node bisa ping satu sama lain

---

## 🛠 2. Tujuan Setup

Dengan panduan ini, kita akan mendapatkan:

✔ Sebuah cluster Kubernetes 3 Nodes  
✔ Worker node siap menjalankan aplikasi  
✔ Jaringan pod menggunakan Calico  
✔ Runtime container menggunakan containerd  

---


## 🔑 3. Persiapan Dasar (Wajib di Semua Node)

Langkah ini dilakukan di:

- `k8s-master`
- `k8s-worker1`
- `k8s-worker2`

### 3.1 Set Hostname

Contoh di masing-masing node:

#### Node Master
```bash
sudo hostnamectl set-hostname k8s-master
```

#### Node Worker 1
```bash
sudo hostnamectl set-hostname k8s-worker1
```

#### Node Worker 2
```bash
sudo hostnamectl set-hostname k8s-worker2
```

---

### 3.2 Tambahkan Hosts Mapping (di Semua Node)

Edit file:
```bash
sudo nano /etc/hosts
```

atau

```bash
sudo vim /etc/hosts
```

Tambahkan baris ini di paling bawah:
```
192.168.1.10 k8s-master
192.168.1.11 k8s-worker1
192.168.1.12 k8s-worker2
```

Tujuannya agar node bisa saling mengenali nama host.

---

### 3.3 Disable Swap

Swap harus dimatikan agar Kubernetes dapat berjalan stabil.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

### 3.4 Aktifkan Kernel Modules

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
```

```bash
sudo modprobe br_netfilter
```

---

### 3.5 Set Sysctl Networking

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```

```bash
sudo sysctl --system
```

Perintah ini mengaktifkan IP forwarding agar Pod bisa berkomunikasi.

---

## 🐳 4. Install containerd (di Semua Node)

containerd adalah container runtime yang direkomendasikan Kubernetes.

### 4.1 Install Dependencies

```bash
sudo apt update
```

```bash
sudo apt install -y curl gnupg2 apt-transport-https ca-certificates
```

---

### 4.2 Tambahkan Repository Docker

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
sudo echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian \
bookworm stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

### 4.3 Install containerd

```bash
sudo apt-get update
```

```bash
sudo apt-get install -y containerd.io
```

---

### 4.4 Generate Default Config

```bash
sudo mkdir -p /etc/containerd
```

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

---

### 4.5 Gunakan Systemd Cgroup Driver

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```
restart containerd, lalu cek status

```bash
sudo systemctl restart containerd
```

```bash
sudo systemctl status containerd
```

> Pastikan status sudah **Running**

---

## 🚀 5. Install Kubernetes (SEMUA NODE)

### 5.1 Tambahkan Repository Kubernetes

```bash
sudo apt-get update
```

```bash
sudo apt-get install -y curl gpg
```

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

### 5.2 Install Package Kubernetes

```bash
sudo apt-get update
```

```bash
sudo apt-get install -y kubeadm kubectl kubelet
```

```bash
sudo apt-mark hold kubeadm kubectl kubelet
```

---

## 🧠 6. Inisialisasi Cluster (Master Node)

Jalankan:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=#ip-master-node
```
> Simpan output **kubeadm join ...**

### 6.1 Konfigurasi kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 🌐 7. Install Calico Tigera Operator (Master Node)

### 7.1 Instalasi Operator

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```

---

### 7.2 Download Konfigurasi Jaringan

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

---

### 7.3 Edit File custom-resource.yaml

```bash
nano custom-resources.yaml
```
> Cari bagian cidr: 192.168.0.0/16 dan **UBAH** menjadi 10.244.0.0/16.

Contoh final: 

```bash
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Perhatikan baris di bawah ini yang diubah:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```
Simpan dan keluar

---

### 7.4 Apply Perubahan dan Konfigurasi Sebelumnya

```bash
kubectl create -f custom-resources.yaml
```

---

### 7.5 Pastikan Pods Calico Sudah Running

```bash
kubectl get pods -n calico-system
```

> Tunggu sampai semua pods running.

> **💡Penting:** Jika pada tahap ini masih belum set worker node, silahkan lakukan set worker mulai dari tahap awal hingga ke tahap 5!

---

## 🤝 8. Join Worker Node ke Master

Di worker node, jalankan command join yang diberikan oleh kubeadm pada bagian 6:

```bash
sudo kubeadm join 192.168.2.104:6443 --token kadmyk.3ehklifrxj4zcamt         --discovery-token-ca-cert-hash sha256:993a57cea7e4fadfab926af60174cc26061dd766766c7de8f5950eb0fbd28651
```

Jika lupa token, jalankan ini

```bash
kubeadm token create --print-join-command
```

> **💡Penting!**

Jika terdapat error seperti di bawah ini:

```bash
[preflight] Running pre-flight checks
[preflight] Some fatal errors occurred:
        [ERROR CRI]: could not connect to the container runtime: failed to create new CRI runtime service: validate service connection: validate CRI v1 runtime API for endpoint "unix:///var/run/containerd/containerd.sock": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService
        [ERROR ContainerRuntimeVersion]: could not connect to the container runtime: failed to create new CRI runtime service: validate service connection: validate CRI v1 runtime API for endpoint "unix:///var/run/containerd/containerd.sock": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService
[preflight] If you know what you are doing, you can make a check non-fatal with --ignore-preflight-errors=...
error: error execution phase preflight: preflight checks failed
To see the stack trace of this error execute with --v=5 or higher
```

Buka file config.toml dengan menjalankan perintah berikut

```bash
nano /etc/containerd/config.toml
```

Cari baris seperti ini:

```bash
disabled_plugins = ["cri"]
```

lalu hapus tulisan "cri" di dalamnya, atau kosongkan saja kurungnya menjadi seperti di bawah ini:

```bash
disabled_plugins = []
```

---

## 🧪 9. Verifikasi Cluster

Cek node yang ada pada cluster

```bash
kubectl get nodes
```

> Pastikan semua node sudah ready dan running.

---


## 🎯 10. Hasil Akhir

Kita sekarang memiliki:

✔ Kubernetes v1.35.0  
✔ 1 Control Plane (Master Node)  
✔ 2 Worker Nodes  
✔ containerd Runtime  
✔ Calico Tigera Networking    

Cluster siap digunakan 🚀

---

## 📎 10. Referensi

- https://kubernetes.io/docs

- https://projectcalico.docs.tigera.io

- https://containerd.io

---