# 🚀 Airpay & iCube — Senior Cloud Operations Engineer Assessment

**Studi Kasus:** Automasi Deployment Magento Open Source pada Minikube dengan Akses Publik HTTPS  
**Arsitektur IaC:** Dual-Layer Configuration (Ansible untuk Server Bootstrapping & K8s Manifests untuk Workloads)  

---

## 📋 1. Prasyarat & Spesifikasi Mesin

### 🖥️ Spesifikasi Mesin Server Lab (Target Deployment)
* **OS:** Ubuntu 24.04.4 LTS (Kernel `6.8.0-134-generic`)
* **CPU:** 2 vCPU (AMD EPYC 7642, x86_64)
* **RAM:** 3.8 GiB Memory + 511 MiB Swap (Diberikan ke Minikube: `3500m`)
* **Disk:** ~79 GB Total Storage (~71 GB Available)
* **Alamat IP Publik:** `172.104.62.55`
* **Metode Akses:** SSH Private Key (`ssh/sk` Ed25519)

### 💻 Prasyarat Mesin Lokal (DevOps Laptop)
* **Ansible:** Versi `>= 2.15` (Untuk mengeksekusi *Configuration Management* Layer 1)
* **Git:** Untuk manajemen kontrol versi repositori
* **Kubectl:** Versi `>= 1.28` (Opsional untuk kendali remote cluster K8s via kubeconfig)
* **Koneksi Internet:** Untuk konektivitas SSH ke server lab dan pemanggilan API DNS Cloudflare

---

## 📦 2. Versi Stack Teknologi (Aplikasi & Datastore)

| Komponen | Pilihan / Versi | Peran & Keterangan |
|---|---|---|
| **Web Application** | **Magento Open Source 2.4.6-p3** | Aplikasi e-commerce utama (PHP-based) |
| **PHP Runtime** | **PHP-FPM 8.1 / 8.2** | Menjalankan logika Magento di dalam Pod Kubernetes |
| **Web Server (In-Pod)**| **Nginx 1.24** | Melayani *static file* dan meneruskan *dynamic request* ke PHP-FPM |
| **Database** | **MySQL 8.0 / MariaDB 10.6** | Datastore relasional produk, pesanan, dan user (StatefulSet + PVC) |
| **Search Engine** | **OpenSearch 2.5** | Mesin pencarian katalog produk (StatefulSet + PVC) |
| **Cache & Session** | **Redis 7.0 (Bonus #2)** | Menjamin stabilitas *session user* saat Magento di-scale multi-replica |
| **Ingress Controller** | **Nginx Ingress K8s Addon** | Router trafik HTTP/S di dalam cluster Minikube |

---

## ⚙️ 3. Cara Menjalankan Minikube & Mengaktifkan Ingress

Inisialisasi cluster Kubernetes dan aktivasi addon dilakukan **secara otomatis dan non-interaktif** menggunakan Ansible Playbooks.

1. **Uji Konektivitas SSH ke Server Lab:**
   ```bash
   cd ansible/
   ansible -i inventories/dev/hosts.ini dev_server -m ping
   ```
2. **Eksekusi Playbook Minikube & Ingress:**
   ```bash
   ansible-playbook -i inventories/dev/hosts.ini playbooks/setup_dev.yml --tags "minikube"
   ```
   *Playbook ini secara otomatis menjalankan:*
   * `minikube start --driver=docker --cpus=2 --memory=3500m`
   * `minikube addons enable ingress` (Mengaktifkan Nginx Ingress Controller K8s)
   * `minikube addons enable metrics-server` (Untuk keperluan pemantauan `kubectl top pods`)

---

## 🚀 4. Cara Deployment dari Awal hingga Aplikasi Siap

Seluruh proses deployment diatur agar *repeatable*, mandiri, dan bebas intervensi manual (sesuai kaidah GitOps):

1. **Kloning Repositori:**
   ```bash
   git clone https://github.com/mrifala29/icube-test.git
   cd icube-test
   ```
2. **Bootstrapping OS & K8s Cluster (Ansible - Layer 1):**
   ```bash
   cd ansible/
   ansible-playbook -i inventories/dev/hosts.ini playbooks/setup_dev.yml
   cd ..
   ```
3. **Deploy Workload Aplikasi & Database (K8s Manifests - Layer 2):**
   Gunakan skrip otomatisasi satu-klik yang menerapkan *Namespace, Secret, ConfigMap, Database, Search Engine, Redis, dan Aplikasi Magento*:
   ```bash
   ./scripts/deploy.sh
   ```
   *Atau jika menggunakan perintah kubectl langsung:*
   ```bash
   kubectl apply -k kubernetes/
   ```

---

## 🌐 5. Cara Mendapatkan URL Publik & Mengakses Admin

Trafik dari internet diarahkan melalui **Cloudflare DNS (HTTPS)** menuju kontainer Nginx Reverse Proxy di host Ubuntu (`172.104.62.55`), yang kemudian mem-forward trafik ke Minikube NodePort Ingress.

* **URL Storefront Publik:** `https://magento.<domain-anda>.com` (Atau IP Server jika diuji via file `hosts` lokal: `http://172.104.62.55`).
* **URL Admin Panel (Custom Secure Path):** `https://magento.<domain-anda>.com/icube-admin` (Demi keamanan, default path `/admin` diubah ke custom path non-default).
* **Mendapatkan Kredensial Admin Portal:**
  ```bash
  # Ambil username admin
  kubectl get secret magento-secret -n magento -o jsonpath='{.data.ADMIN_USERNAME}' | base64 -d
  echo ""
  # Ambil password admin
  kubectl get secret magento-secret -n magento -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
  echo ""
  ```

---

## 🧪 6. Cara Verifikasi Fungsi Utama

### A. Verifikasi Status Kesehatan Pod & Cluster
Pastikan seluruh komponen berstatus **Running** dan *Readiness/Liveness Probe* berhasil:
```bash
kubectl get pods -n magento
kubectl get svc,ingress -n magento
kubectl top pods -n magento
```

### B. Uji Persistensi Data (Self-Healing & Failover Demo)
Simulasikan kerusakan atau penghapusan Pod database untuk membuktikan data toko tidak hilang:
```bash
# 1. Hapus paksa pod MySQL
kubectl delete pod <nama-pod-mysql> -n magento

# 2. Amati proses otomatisasi self-healing K8s
kubectl get pods -n magento -w

# 3. Verifikasi data di storefront tetap utuh setelah pod baru berstatus Running
```

### C. Uji Keamanan Session Multi-Replica (Redis Proof)
Simulasikan pembebanan trafik dengan meningkatkan jumlah replika Magento web server:
```bash
# 1. Scale deployment menjadi 2 replika
kubectl scale deployment magento --replicas=2 -n magento

# 2. Buka browser, login ke admin panel /icube-admin
# 3. Lakukan refresh berkali-kali untuk membuktikan load balancer membagi trafik ke pod berbeda tanpa membuat session logout berkat integrasi Redis Cache.
```

---

## 💾 7. Cara Backup & Restore

Repositori ini menyediakan skrip otomatisasi di direktori `scripts/` untuk melindungi data transaksional dan media toko.

* **Melakukan Backup Otomatis (Database Dump + Media Tarball):**
  ```bash
  ./scripts/backup.sh
  ```
  *Output backup akan disimpan secara terstruktur di folder `backups/<timestamp>/` berisikan file `.sql` dan arsip `.tar.gz` untuk folder `/pub/media`.*
* **Melakukan Restore Data:**
  ```bash
  ./scripts/restore.sh backups/<timestamp_folder>/
  ```

---

## 🧹 8. Cara Cleanup Environment

Untuk membersihkan seluruh *workload* aplikasi tanpa merusak sistem operasi server lab:

* **Cleanup Workloads Magento saja (Menghapus Pod, Service, & PVC):**
  ```bash
  ./scripts/cleanup.sh
  # Atau secara manual: kubectl delete -k kubernetes/
  ```
* **Cleanup Total Cluster Minikube (Factory Reset K8s):**
  ```bash
  minikube delete --all
  ```

---

## ⚠️ 9. Known Issues, Limitation, & Asumsi yang Digunakan

### 🐞 Known Issues
*(Dikosongkan / Akan dilampirkan berdasarkan temuan log saat tahap eksekusi pengujian di lapangan)*

### 🚧 Limitations
*(Dikosongkan / Akan dilampirkan setelah pengukuhan benchmark beban)*

### 📌 Asumsi yang Digunakan
1. **Cloudflare DNS Integration:** Manajemen sertifikat TLS/SSL dan resolusi DNS dikelola pada UI Cloudflare Dashboard (A Record menunjuk ke IP `172.104.62.55` dengan status *Proxied / Orange Cloud* diaktifkan).
2. **Resource Constraint Engineering:** Mengingat kapasitas RAM server lab terbatas pada 4GB, alokasi memori (`resources.requests` dan `resources.limits`) pada *StatefulSet* MySQL, OpenSearch, dan PHP-FPM diatur secara ketat dan presisi agar tidak memicu OOMKilled oleh Linux Kernel.
3. **Portabilitas Cloud-Native:** StorageClass yang digunakan di lingkungan Minikube adalah `standard` (berbasis `hostPath`). Dalam skenario migrasi ke production AWS EKS, spesifikasi manifest ini dirancang 100% portabel dan hanya membutuhkan pergantian nama StorageClass menjadi `gp3` (AWS EBS).
4. **Isolasi Keamanan (Zero-Trust):** Akses komunikasi ke *database* dan *search engine* dilarang keras dari luar cluster, dan dibatasi hanya dari Pod Magento dalam *Namespace* yang sama melalui *NetworkPolicy*.

---

## ⏱️ 10. Estimasi Waktu Pengerjaan

Total estimasi waktu penyelesaian proyek ini dari desain arsitektur hingga produksi adalah **12 Jam**, yang dialokasikan secara modular menjadi **6 Jam per Hari** dalam 2 hari kerja:

* **Hari Ke-1 (6 Jam - Infrastruktur & Datastore):** 
  * Automasi Ansible (Layer 1 Bootstrapping, Docker, Minikube, Nginx Proxy).
  * Inisialisasi Kubernetes Base (Namespace, PVC, Secret, ConfigMap).
  * Deployment StatefulSet Datastore (MySQL, OpenSearch, dan Redis Cache).
* **Hari Ke-2 (6 Jam - Workload Aplikasi, Hardening, & Demo Preparation):**
  * Deployment Magento App & Nginx Ingress Controller routing.
  * Implementasi skrip otomatisasi (`deploy.sh`, `backup.sh`, `restore.sh`, `cleanup.sh`).
  * Uji ketahanan failover (*pod deletion* & *multi-replica scaling*).
  * Finalisasi dokumentasi README.md dan pengumpulan bukti screenshot/log (`evidence/`).
