# SIEM — Security Information & Event Management

> NCC Laboratory 2026 - Open Recruitment Project  
> https://llama-chat.my.id/siem

---

## 📋 Daftar Isi

- [Gambaran Umum](#gambaran-umum)
- [Arsitektur Sistem](#arsitektur-sistem)
- [Teknologi yang Digunakan](#teknologi-yang-digunakan)
- [Struktur Direktori](#struktur-direktori)
- [CI/CD Pipeline](#cicd-pipeline)
- [Tim Engineer](#tim-engineer)

---

## Gambaran Umum

Sistem SIEM ini dirancang untuk **mengumpulkan, mem-parsing, menyimpan, dan memvisualisasikan log** dari berbagai sumber secara *real-time*. Dibangun di atas **Docker Compose**, dilengkapi pipeline **Jenkins CI/CD** dan *quality gate* **SonarQube**, serta dapat diakses publik melalui VPS dengan reverse proxy Nginx.

Sistem ini terdiri dari beberapa komponen utama:

| Komponen | Fungsi |
|---|---|
| **Log Collector** | Membaca file log dari path yang dikonfigurasi via dashboard (Python/Go service) |
| **Log Parser** | Mengekstrak field penting: `timestamp`, `level`, `source`, `message`; mendukung plugin per format (syslog, nginx, JSON) |
| **Rule Engine** | Mengevaluasi rule berbasis JSON/YAML dan menetapkan severity (`INFO` / `WARN` / `CRITICAL`) |
| **Alert & Notifikasi** | Mengirim notifikasi email/webhook saat rule terpenuhi, menyimpan alert ke database |
| **PostgreSQL** | Menyimpan events, alerts, konfigurasi rule, dan riwayat log yang masuk |
| **WebSocket Server** | Push event real-time ke dashboard tanpa polling (Node.js / FastAPI) |
| **Dashboard UI** | React/Next.js — monitoring live, input path log, manajemen rule, badge severity |
| **CI/CD + DevOps** | Jenkins pipeline, SonarQube quality gate, Docker Compose, deploy ke VPS publik |

---

## Arsitektur Sistem

```
Log Files
    │
    ▼
┌─────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Log Collector  │────▶│   Log Parser     │────▶│  Rule Engine     │
│  (Python/Go)    │      │  (syslog/nginx/  │      │  (JSON/YAML)     │
└─────────────────┘      │   JSON plugin)   │      └────────┬─────────┘
                         └──────────────────┘               │
                                 │                          ▼
                                 ▼                ┌──────────────────┐
                        ┌──────────────────┐      │  Alert & Notif   │
                        │   PostgreSQL     │◀────│  (Email/Webhook) │
                        │   (events,       │      └──────────────────┘
                        │    alerts,rules) │              │
                        └────────┬─────────┘              │
                                 │                        │
                                 ▼                        ▼
                        ┌──────────────────────────────────────────┐
                        │         WebSocket Server                 │
                        └──────────────────┬───────────────────────┘
                                           │
                                           ▼
                        ┌──────────────────────────────────────────┐
                        │         Dashboard UI (Next.js)           │
                        │  Live Feed │ Rule Builder │ Alert Mgmt   │
                        └──────────────────────────────────────────┘
                                           │
                                     Nginx (HTTPS)
                                           │
                                     VPS Publik
```

---

## Teknologi yang Digunakan

| Layer | Teknologi |
|---|---|
| Backend / Collector | Go |
| Frontend | React / Next.js 14 + Tailwind CSS |
| Database | PostgreSQL 16 |
| Real-time | WebSocket (ws / socket.io atau FastAPI) |
| Containerisasi | Docker + Docker Compose |
| CI/CD | Jenkins (BlueOcean) |
| Code Quality | SonarQube |
| Reverse Proxy | Nginx | 
| Notifikasi | Discord Webhook + GitHub |

---

## Struktur Direktori

```
.
├── backend/
│   ├── collector/          # Log Collector Service
│   ├── parser/             # Log Parser + plugin format
│   │   ├── plugins/
│   │   │   ├── syslog.py
│   │   │   ├── nginx.py
│   │   │   └── json_passthrough.py
│   ├── api/                # REST API (events, alerts, rules)
│   ├── websocket/          # WebSocket server
│   ├── rules/              # Rule Engine + evaluator
│   └── Dockerfile
├── frontend/
│   ├── app/                # Next.js App Router
│   │   ├── dashboard/      # Halaman monitoring real-time
│   │   ├── rules/          # Rule Builder UI
│   │   ├── alerts/         # Alert Management
│   │   └── search/         # Pencarian & filter log
│   ├── components/
│   └── Dockerfile
├── database/
│   └── init/
│       └── 001_init.sql    # Schema: events, alerts, rules, log_sources
├── nginx/
│   ├── nginx.conf
│   └── certs/
├── jenkins/
│   └── Dockerfile
├── docker-compose.yml
├── docker-compose.jenkins-sonarqube.yml
├── Jenkinsfile
├── .env.example
└── README.md
```

---

## CI/CD Pipeline

Pipeline Jenkins berjalan otomatis setiap kali ada push ke repository.

```
Checkout → Setup → Backend Build & Test → Frontend Install
    → Frontend Lint → Frontend Build → SonarQube Analysis → Quality Gate
```

| Service | URL |
|---|---|
| Jenkins | `http://127.0.0.1:8080/jenkins` |
| SonarQube | `http://127.0.0.1:9000/sonarqube` |

### Quality Gate

- Coverage unit test **≥ 70%** untuk modul backend
- **Zero** blocker/critical issue di SonarQube
- Pipeline otomatis **gagal** jika quality gate tidak terpenuhi

---

## Pembagian Tugas

### E1 — Mochammad Irfan Sandy
**Backend & Data Engineer**

Bertanggung jawab atas seluruh lapisan data dan backend service, mulai dari pengumpulan log hingga API yang dikonsumsi frontend.

**Job Desk:**
- Membangun **Log Collector Service** — membaca file log dari path yang dikonfigurasi, mendeteksi baris baru secara polling/inotify
- Mengembangkan **Log Parser** beserta plugin format: syslog (RFC 5424 & BSD), nginx access log, dan JSON passthrough
- Merancang **skema PostgreSQL** dan mengelola migrasi database (tabel `events`, `alerts`, `rules`, `log_sources`)
- Membuat **Storage Service** — menyimpan hasil parsing ke PostgreSQL dengan latensi < 2 detik
- Membangun **REST API**: `GET/POST /events`, `GET /events/:id` (pagination + filter severity), serta CRUD `/rules`
- Membangun **WebSocket Server** — push event baru ke semua client yang terkoneksi secara real-time (< 1 detik)
- Menulis **unit test** untuk parser (syslog, nginx, JSON) dan API endpoint (coverage ≥ 70%)
- Menambahkan **plugin parser custom** untuk format JSON aplikasi dengan field bebas

---

### E2 — Tuti Purwaningsih
**Rule Engine & Notification Engineer**

Bertanggung jawab atas sistem pendeteksian ancaman, evaluasi aturan keamanan, dan pengiriman notifikasi.

**Job Desk:**
- Merancang **skema rule JSON/YAML** — field: `name`, `condition`, `threshold`, `severity`, `action`
- Membangun **Rule Evaluator**:
  - *Threshold count* — contoh: 5 error dalam 2 menit (deteksi brute-force)
  - *Pattern matching* — regex pada field `message`
  - *Kombinasi kondisi* — AND/OR antar field
- Menentukan dan mengimplementasikan **assignment severity**: `INFO` / `WARN` / `CRITICAL` per rule
- Implementasi **alert deduplication** — mencegah alert duplikat untuk event yang sama dalam window 5 menit
- Mengelola **alert lifecycle**: status `open` → `acknowledged` → `resolved`
- Menyiapkan **built-in rules** siap pakai: brute-force SSH, repeated 4xx/5xx, keyword `CRITICAL`/`ERROR`
- Menulis **unit test** untuk rule evaluator (coverage ≥ 70%)

---

### E3 — Lucky Himawan Prasetya
**Frontend & DevOps Engineer**

Bertanggung jawab atas tampilan dashboard, pengalaman pengguna, serta seluruh infrastruktur deployment dan pipeline CI/CD.

**Job Desk:**

*Frontend:*
- Membangun **scaffold Dashboard UI** dengan Next.js 14 + Tailwind CSS (routing, layout, halaman utama)
- Mengembangkan **halaman monitoring real-time** — tabel event via WebSocket, badge severity berwarna (`INFO`/`WARN`/`CRITICAL`)
- Membuat **form input path log** — tambah/hapus sumber log tanpa restart container
- Mengembangkan **halaman pencarian log** — filter severity, source
- Membangun **Rule Builder UI** — form buat/edit rule dengan preview kondisi, perubahan langsung aktif
- Membuat **halaman Alert Management** — list alert, status open/ack/resolved, filter severity, update real-time via WebSocket
- Fitur **export log** ke CSV/JSON dari halaman pencarian
- UI **konfigurasi notifikasi per-rule** — toggle email/webhook, perubahan tersimpan langsung efektif

*DevOps & Infrastruktur:*
- Membuat **Dockerfile** untuk semua service (base alpine, ukuran minimal)
- Mengelola **Docker Compose** — orchestrasi semua service (backend, frontend, DB, Nginx, WebSocket)
- Setup dan konfigurasi **Jenkins** + pipeline: lint → test → build image → push registry
- Integrasi **SonarQube** ke Jenkins dengan quality gate
- **Deploy ke VPS publik** — Docker Compose production, env vars dari `.env`, Nginx reverse proxy (HTTPS)
- Mengembangkan sistem **notifikasi webhook** — HTTP POST ke endpoint yang dikonfigurasi, payload JSON valid

---

---

*NCC Laboratory 2026 · Final Project · Kelompok 3*
