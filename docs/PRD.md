# PRD — Senior Cloud Operations Engineer Assessment
**Studi Kasus:** Deploy Magento Open Source di Minikube + Akses Publik HTTPS  
**Batas waktu:** Maksimal 3 hari kalender  
**Demo:** 45 menit demo + 30 menit diskusi teknis | Skor maksimal: 100 poin

---

## 🎯 Inti Task (Satu Kalimat)

> Deploy Magento lengkap (web, DB, search engine) di **Minikube** pada server lab, bisa diakses publik via **HTTPS**, deployment **bisa diulang dari Git** tanpa konfigurasi manual.

---

## 📋 Task Inti (Wajib)

### A. Komponen Aplikasi yang Harus Berjalan

| # | Komponen | Pilihan | Keterangan |
|---|---|---|---|
| 1 | **Web Application** | Magento Open Source | Versi bebas, sesuaikan PHP & Search Engine |
| 2 | **Web Server** | Nginx atau Apache | Jalan di dalam pod K8s bersama PHP-FPM |
| 3 | **PHP-FPM** | Sesuai versi Magento | Satu pod dengan web server |
| 4 | **Database** | MySQL atau MariaDB | Harus pakai PVC agar data persisten |
| 5 | **Search Engine** | OpenSearch atau Elasticsearch | Harus kompatibel versi Magento, pakai PVC |
| 6 | **Ingress Controller** | Nginx Ingress (addon Minikube) | Router trafik HTTP/S di dalam cluster K8s |

---

### B. Kubernetes Resources yang Harus Ada

| Resource | Fungsi | Catatan |
|---|---|---|
| **Namespace** | Isolasi workload | Dedicated namespace `magento` |
| **Deployment** | Menjalankan pod Magento | Multi-replica ready |
| **StatefulSet** | DB & Search Engine | Lebih tepat untuk workload stateful |
| **Service (ClusterIP)** | Komunikasi antar pod | DB & Search tidak boleh diakses dari internet |
| **Ingress** | Routing trafik dari luar cluster | Arahkan ke Service Magento |
| **ConfigMap** | Konfigurasi non-sensitif | Base URL, app mode, env vars |
| **Secret** | Konfigurasi sensitif | DB password, admin credential — tidak boleh commit ke Git |
| **PVC** | Storage persisten | Wajib untuk DB, Search Engine, dan shared media |
| **readinessProbe** | Pod siap terima trafik | Cegah 502/503 saat pod baru start |
| **livenessProbe** | Pod masih sehat | Restart otomatis jika hang |
| **resources.requests & limits** | Alokasi CPU/Memory per pod | Server lab 4GB RAM — harus cermat |

---

### C. Klarifikasi: "Nginx" Ada 3 Konteks Berbeda

| Konteks | Lokasi | Di-setup oleh |
|---|---|---|
| **Nginx web server Magento** | Di dalam pod K8s | K8s manifest |
| **Nginx Ingress Controller** | Di dalam cluster K8s (pod tersendiri) | `minikube addons enable ingress` — bukan Ansible |
| **Nginx reverse proxy host** | Di dalam kontainer Docker pada host Ubuntu | Ansible (Docker Compose) — routing traffic internet → Minikube |

---

### D. Acceptance Criteria — Magento Dianggap Berhasil Jika

- [ ] Storefront bisa dibuka dari URL publik HTTPS (bukan localhost, bukan IP Minikube)
- [ ] Admin panel bisa dibuka via **custom path non-default** (bukan `/admin`, misal `/icube-admin`)
- [ ] Magento terhubung ke DB dan Search Engine
- [ ] Static content (CSS, JS, gambar) tampil normal
- [ ] Cache Magento aktif
- [ ] Minimal ada **1 produk** yang bisa dibuka dari storefront
- [ ] Base URL Magento sudah pakai domain publik

---

### E. Public Access & HTTPS

**Masalah:** Minikube berjalan sebagai cluster virtual di dalam server. Traffic internet tidak bisa langsung masuk — perlu "jembatan" dari IP publik server → Minikube.

**Solusi yang digunakan: Nginx reverse proxy (di dalam Docker) + Cloudflare DNS**

Ini workflow yang bersih dan terisolasi: jalankan Nginx sebagai kontainer Docker di host Ubuntu (via Docker Compose), set A record di Cloudflare dashboard ke IP `172.104.62.55`, Cloudflare handle HTTPS. Nginx kontainer mem-forward traffic ke Minikube NodePort.

```
Browser → Cloudflare (HTTPS/TLS) → 172.104.62.55:80 → Nginx Container (Docker di Host OS)
                                                          ↓ proxy_pass
                                                    Minikube NodePort
                                                          ↓
                                               Nginx Ingress Controller (pod)
                                                          ↓
                                                   Service → Pod Magento
```

**Alternatif jika tidak ingin install Nginx di host OS:**
- Cloudflare Tunnel (`cloudflared` daemon) — install via Ansible, konfigurasi di CF Zero Trust UI, tidak perlu atur DNS manual
- Ngrok — cepat setup tapi URL berubah-ubah

**Syarat apapun pilihannya:**
- HTTP otomatis redirect ke HTTPS
- DB dan Search Engine tidak terekspos ke internet (ClusterIP)
- Token tunnel / credential tidak masuk Git

---

### F. Persistence & Recovery (Diuji Saat Demo)

Data harus tetap ada setelah pod dihapus dan dibuat ulang:
- Data database Magento (products, orders, customers)
- Media produk & file upload
- Konfigurasi aplikasi persisten

**Simulasi:**
```bash
kubectl delete pod <nama-pod-mysql> -n magento
kubectl get pods -n magento -w                          # pod baru muncul otomatis
kubectl exec -it <pod-baru> -n magento -- mysql -u root -p -e "SHOW DATABASES;"
```

---

### G. Skenario Troubleshooting (5 Skenario — Harus Bisa Dijelaskan)

| # | Skenario | Langkah Investigasi |
|---|---|---|
| 1 | **HTTP 502** | `kubectl logs <pod>` → PHP-FPM error; `kubectl describe ingress` → verifikasi backend; bedakan upstream timeout vs port salah vs PHP-FPM crash |
| 2 | **HTTP 503** | `kubectl get endpoints -n magento` → endpoint kosong? readiness probe gagal? DB/Search Engine mati? |
| 3 | **Pod OOMKilled** | `kubectl describe pod` → lihat Last State OOMKilled; `kubectl top pods -n magento`; kurangi PHP-FPM worker atau naikkan limit |
| 4 | **Database pod terhapus** | `kubectl get pvc -n magento` → PVC masih Bound; buat pod baru → data tetap ada; jelaskan lifecycle PVC vs Pod |
| 5 | **Search engine tidak sehat** | `kubectl exec` → curl health endpoint; baca log cluster health; dampak: produk tidak bisa dicari, checkout bisa error |

---

## ⭐ Task Opsional (Bonus — Nilai Tambah)

| # | Bonus Item | Alasan |
|---|---|---|
| 1 | **Helm Chart / Kustomize** | Manifest terstruktur, value bisa dioverride per environment |
| 2 | **Redis (Cache & Session)** | **Sangat direkomendasikan** — tanpa Redis, scale ke 2 replika = session hilang |
| 3 | **HorizontalPodAutoscaler (HPA)** | Auto-scaling berdasarkan CPU/Memory |
| 4 | **PodDisruptionBudget (PDB)** | Jamin pod minimal tersedia saat maintenance cluster |
| 5 | **NetworkPolicy** | DB & Search Engine hanya bisa diakses dari pod Magento |
| 6 | **Backup & Restore otomatis** | CronJob K8s untuk backup DB dump + media |
| 7 | **CI/CD Pipeline** | GitHub Actions: lint manifest, scan image, auto-deploy |
| 8 | **TLS otomatis** | cert-manager + Let's Encrypt |
| 9 | **Custom container image** | Dockerfile Magento yang lean dan minimal |
| 10 | **Image vulnerability scanning** | Trivy / Snyk scan CVE sebelum deploy |
| 11 | **Observability stack** | Prometheus + Grafana + Loki |
| 12 | **Zero-downtime deployment** | RollingUpdate `maxSurge`/`maxUnavailable` + graceful shutdown |

---

## 🏗️ Arsitektur IaC

### Layer 1 — Ansible (Server Bootstrapping & Configuration Management)

Setup OS server lab dari laptop via SSH secara otomatis, modular, dan terstruktur tanpa intervensi manual. Menggunakan pendekatan **Role-Based Architecture** dengan fitur tagging agar instalasi dapat dilakukan secara keseluruhan ataupun parsial.

#### 1. Review & Ruang Lingkup Instalasi
Berdasarkan kebutuhan deployment Minikube dan observabilitas saat demo, Ansible bertanggung jawab menginstal dan mengonfigurasi komponen berikut di host OS Ubuntu:

*   **Base & Debugging Tools (`base/common`)**: Package dasar untuk monitoring dan debugging (`git`, `curl`, `wget`, `vim`, `htop`, `jq`, `unzip`, `dnsutils`, `net-tools`, serta **`k9s`**). Cukup menggunakan file `tasks/main.yml`.
*   **Container Engine (`base/docker`)**: Docker Engine, CLI, serta `docker-compose-plugin`. Cukup menggunakan file `tasks/main.yml` dan `handlers/main.yml`.
*   **Orchestration Ecosystem (`orchestration/minikube`)**: Binary Minikube yang dijalankan di dalam Docker (`minikube start --driver=docker`), `kubectl`, `helm`, `k9s`, inisialisasi cluster sesuai spesifikasi `group_vars`, aktivasi addon (`ingress`, `metrics-server`), serta tuning kernel Linux (`sysctl` rules di `tasks/main.yml`). Dilengkapi `defaults/main.yml` dan `templates/` (misalnya untuk template skrip environment aliases `alias k=kubectl` dan auto-completion).
*   **Networking & Reverse Proxy (`networking/nginx_proxy`)**: Nginx reverse proxy yang dijalankan **di dalam kontainer Docker** (menggunakan Docker Compose dan template konfigurasi `.j2`) untuk meneruskan trafik HTTPS publik dari Cloudflare ke Minikube NodePort agar host OS tetap bersih tanpa instalasi Nginx langsung di sistem operasi.

> [!IMPORTANT]
> **Review Mengenai Redis:**  
> **Redis tidak diinstall di host OS melalui Ansible.** Dalam arsitektur cloud-native proyek ini, Redis di-deploy sebagai **Pod di dalam cluster Kubernetes** (`kubernetes/datastore/redis.yaml`).
> Alasan arsitektural:
> 1. Magento running di dalam K8s Pod berkomunikasi secara lokal via DNS cluster (`redis.magento.svc.cluster.local`) dengan latensi minimal.
> 2. Menjaga portabilitas saat migrasi ke production (AWS EKS), di mana Redis di K8s nantinya dapat diganti ke AWS ElastiCache tanpa mengubah konfigurasi host OS.

---

#### 2. Struktur Direktori Modular
Struktur direktori dirancang rapi dengan pemisahan `inventories`, `group_vars`, dan kategorisasi `roles` (dilengkapi `defaults`, `tasks`, `handlers`, dan `templates`).

```text
ansible/
├── ansible.cfg                         # SSH key path, remote user, pipelining=true
├── inventories/
│   └── dev/                            # Environment Dev Assessment
│   │   ├── hosts.ini                   # [dev_server] root@172.104.62.55
│   │   └── group_vars/
│   │       └── all.yml                 # Global vars (minikube_cpus, domain, ports)
├── playbooks/
│   └── setup_dev.yml                   # Entrypoint utama untuk Environment Dev
└── roles/
    ├── base/
    │   ├── common/                     # Basic tools (git, curl, jq, htop, k9s)
    │   │   └── tasks/main.yml          # Cukup tasks saja
    │   └── docker/                     # Docker Engine & Compose plugin
    │       ├── tasks/main.yml          # Cukup tasks & handlers
    │       └── handlers/main.yml       # Restart Docker service handler
    ├── orchestration/
    │   └── minikube/                   # Minikube (driver=docker), kubectl, helm, k9s
    │       ├── defaults/main.yml       # Default resource limit (CPU: 2, RAM: 3500m, driver: docker)
    │       ├── tasks/main.yml          # Sysctl & instalasi cluster/tools
    │       └── templates/
    │           └── k8s-env.sh.j2       # Template profile aliases & bash auto-completion
    └── networking/
        └── nginx_proxy/                # Nginx reverse proxy running in Docker container
            ├── defaults/main.yml
            ├── tasks/main.yml
            ├── handlers/main.yml       # Restart/reload Nginx docker container
            └── templates/
                ├── docker-compose.yml.j2 # Template deployment Nginx di Docker Compose
                └── nginx-magento.conf.j2 # Template reverse proxy host → NodePort
```

---

#### 3. Strategi Eksekusi & Penggunaan Tags
Playbook entrypoint (`setup_dev.yml`) memanggil seluruh role yang dikelompokkan dengan `tags`. Ini memungkinkan kita mengeksekusi instalasi secara spesifik atau per bagian.

**Contoh Desain Playbook (`playbooks/setup_dev.yml`):**
```yaml
---
- name: Setup Base OS Requirements & Tools
  hosts: dev_server
  become: true
  roles:
    - role: base/common
      tags: ['base', 'common']
    - role: base/docker
      tags: ['base', 'docker']

- name: Setup Kubernetes Cluster (Minikube & Tools)
  hosts: dev_server
  become: true
  roles:
    - role: orchestration/minikube
      tags: ['orchestration', 'minikube', 'k8s']

- name: Setup Networking & Reverse Proxy
  hosts: dev_server
  become: true
  roles:
    - role: networking/nginx_proxy
      tags: ['networking', 'proxy', 'nginx']
```

**Cara Penggunaan (CLI Eksekusi):**
```bash
cd ansible/
# 1. Uji konektivitas ke server dev
ansible -i inventories/dev/hosts.ini dev_server -m ping

# 2. Eksekusi keseluruhan instalasi dari awal sampai siap
ansible-playbook -i inventories/dev/hosts.ini playbooks/setup_dev.yml

# 3. Eksekusi parsial hanya bagian Docker / Base OS
ansible-playbook -i inventories/dev/hosts.ini playbooks/setup_dev.yml --tags "docker"

# 4. Eksekusi parsial hanya instalasi Minikube & K8s tools
ansible-playbook -i inventories/dev/hosts.ini playbooks/setup_dev.yml --tags "minikube"

# 5. Eksekusi parsial update konfigurasi Nginx proxy saja
ansible-playbook -i inventories/dev/hosts.ini playbooks/setup_dev.yml --tags "proxy"
```

---

### Layer 2 — Kubernetes Manifests (Workload Deployment)

File YAML yang mendefinisikan semua resource K8s. **File ini portable ke EKS** — perbedaan saat migrasi hanya di `StorageClass` dan `Service type LoadBalancer`.

```bash
kubectl apply -f kubernetes/base/        # Namespace, ConfigMap, Secret, PVC
kubectl apply -f kubernetes/datastore/   # MySQL, OpenSearch, Redis (bonus)
kubectl apply -f kubernetes/app/         # Magento Deployment, Service, Ingress
kubectl apply -f kubernetes/addons/      # Bonus: HPA, PDB, NetworkPolicy
```

**Perbedaan kecil saat pindah ke EKS:**

| Aspek | Minikube | EKS |
|---|---|---|
| StorageClass | `standard` (hostPath) | `gp2` / `gp3` (EBS) |
| LoadBalancer | Perlu `minikube tunnel` | Otomatis dapat ELB/ALB |
| Public access | Nginx host / Cloudflare Tunnel | AWS ALB Ingress Controller |
| TLS | Cloudflare / manual | cert-manager + ACM |

---

## 📂 Struktur Direktori Repository

```text
.
├── README.md                               # Prasyarat, cara deploy step-by-step, verify, cleanup
├── PRD.md                                  # Dokumen ini
├── connect.sh                              # Quick SSH ke server lab
├── .gitignore                              # Ignore: ssh/sk, .env, secret.yaml, *.pem
│
├── architecture/
│   ├── architecture-diagram.png            # Diagram: Browser → CF → Nginx → Ingress → Pod → DB
│   └── traffic-flow.md                     # Penjelasan alur request end-to-end
│
├── ansible/                                # IaC Layer 1 — Server Bootstrapping & Configuration
│   ├── ansible.cfg
│   ├── inventories/
│   │   └── dev/
│   │       ├── hosts.ini                   # Inventory server dev
│   │       └── group_vars/
│   │           └── all.yml                 # Variabel global (CPU, memori, domain)
│   ├── playbooks/
│   │   └── setup_dev.yml                   # Entrypoint playbook dengan fitur tags
│   └── roles/
│       ├── base/
│       │   ├── common/                     # Debugging tools (cukup tasks/main.yml)
│       │   └── docker/                     # Docker Engine (cukup tasks & handlers)
│       ├── orchestration/
│       │   └── minikube/                   # Minikube (driver=docker), kubectl, helm, k9s
│       └── networking/
│           └── nginx_proxy/                # Nginx reverse proxy running in Docker container
│
├── kubernetes/                             # IaC Layer 2 — K8s Workloads (portable ke EKS)
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.example.yaml             # Template — nilai asli jangan commit!
│   │   └── pvc.yaml
│   ├── datastore/
│   │   ├── database.yaml                   # MySQL/MariaDB StatefulSet + Service
│   │   ├── search-engine.yaml              # OpenSearch/Elasticsearch StatefulSet + Service
│   │   └── redis.yaml                      # [Bonus] Redis Deployment + Service
│   ├── app/
│   │   ├── magento.yaml                    # Deployment Magento (Nginx + PHP-FPM)
│   │   ├── service.yaml                    # ClusterIP Service
│   │   └── ingress.yaml                    # Ingress rules + TLS
│   └── addons/                             # [Bonus]
│       ├── hpa-pdb.yaml
│       └── network-policy.yaml
│
├── scripts/
│   ├── deploy.sh                           # Full deploy: ansible-playbook → kubectl apply
│   ├── verify.sh                           # Cek pod running, curl URL publik
│   ├── backup.sh                           # mysqldump + rsync media
│   ├── restore.sh
│   └── cleanup.sh                          # kubectl delete + minikube stop
│
├── ssh/                                    # Local only — masuk .gitignore
│   ├── sk                                  # Private key Ed25519 (chmod 600)
│   └── server-info.json
│
├── .github/                                # [Bonus] CI/CD
│   └── workflows/
│       └── deploy-pipeline.yml
│
└── docs/
    ├── troubleshooting.md                  # 5 skenario: 502, 503, OOM, DB pod, Search
    ├── production-design.md                # Roadmap migrasi ke EKS
    └── evidence/                           # Screenshot: storefront, admin, kubectl output
```

---

## 🧪 Final Checklist

| Area | Kriteria |
|---|---|
| **Public HTTPS** | Storefront via HTTPS, HTTP redirect otomatis |
| **Magento berfungsi** | Homepage, produk, admin (custom path) bisa dibuka |
| **Pod sehat** | `kubectl get pods -n magento` → semua `Running` |
| **Data persisten** | Data tetap ada setelah `kubectl delete pod` pada pod DB |
| **Secret aman** | Tidak ada credential aktif di Git |
| **Repeatable** | Clone repo → ikuti README → berhasil tanpa petunjuk lisan |
| **Pemahaman arsitektur** | Bisa jelaskan alur trafik, kenapa tiap resource ada, failure mode |

---

## 🖥️ Spesifikasi Server Lab (Verified)

| Parameter | Nilai | Dampak |
|---|---|---|
| **IP Publik** | `172.104.62.55` | Port 80 & 443 bebas |
| **OS** | Ubuntu 24.04.4 LTS, Kernel `6.8.0-134-generic` | Kompatibel Docker & Minikube |
| **CPU** | 2 vCPU (AMD EPYC 7642, x86_64) | `minikube start --cpus=2` |
| **RAM** | 3.8 GiB + 511 MiB Swap | `minikube start --memory=3500m` — sisakan untuk OS |
| **Disk** | 79 GB total, ~71 GB free | Cukup untuk image + PVC |
| **Virtualisasi** | Full KVM | Pakai driver `docker` untuk Minikube |

---

## ⚡ Urutan Eksekusi Modular (Tahap / Kategori Kerja)

Agar pengerjaan terstruktur, mudah dilacak, dan tidak membebani memori/konteks AI di sesi chat baru, pengerjaan dibagi menjadi **6 Tahap Modular**. Di setiap sesi chat baru, Anda cukup menyebutkan tahap mana yang sedang dikerjakan (misal: *"Ayo kerjakan Tahap 1"*):

| Urutan Tahap | Kategori / Fokus Kerja | Deliverables & File IaC yang Dibuat | Target Verifikasi (Selesai Jika...) |
|---|---|---|---|
| **Tahap 1** | **Server Bootstrapping (IaC Layer 1)** | • Folder `ansible/` modular (roles, inventories, group_vars)<br>• Playbook entrypoint (`setup_dev.yml`) dengan dukungan `--tags` | Server dev bersiap otomatis dari laptop via Ansible; `minikube status` berstatus **Running**. |
| **Tahap 2** | **K8s Base & Datastore (IaC Layer 2)** | • Manifest: `namespace.yaml`, `secret.yaml`, `pvc.yaml`<br>• Deploy MySQL, OpenSearch, & **Redis (Bonus #2)** | Pod database, search engine, dan redis berstatus **Running** dengan volume (PVC) terikat (*Bound*). |
| **Tahap 3** | **Magento Web & Public HTTPS** | • Manifest: `magento.yaml`, `service.yaml`, `ingress.yaml`<br>• Konfigurasi Nginx Host Proxy + Cloudflare DNS | Storefront Magento & Admin Portal (`/icube-admin`) bisa dibuka lancar via HTTPS di browser. |
| **Tahap 4** | **Script Otomatisasi & Bonus IaC** | • Script: `deploy.sh`, `backup.sh`, `restore.sh` **(Bonus #6)**<br>• `kustomization.yaml` **(Bonus #1)** & `network-policy.yaml` **(Bonus #5)** | Skrip satu-klik berfungsi sukses; NetworkPolicy aktif memblokir akses ilegal ke DB (Zero-Trust). |
| **Tahap 5** | **Pengujian Ujian (Self-Healing & Scaling)** | • Simulasi tes failover dan reliability | Uji `delete pod` MySQL (data bukti tidak hilang); Uji `scale replicas=2` (session bukti tidak putus). |
| **Tahap 6** | **Finalisasi Dokumen & Persiapan Demo** | • Lengkapi `README.md` & folder bukti (`evidence/`)<br>• Review jawaban untuk 5 Skenario Troubleshooting | Repository siap di-push ke Git; siap dan percaya diri menghadapi demo 45 menit + diskusi teknis. |


