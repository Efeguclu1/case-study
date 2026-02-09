# Mimari Dokumantasyon

## Genel Bakis

Bu proje, bir MERN (MongoDB, Express.js, React, Node.js) stack uygulamasini ve Python ETL projesini Google Kubernetes Engine (GKE) uzerine deploy eder.

## Teknoloji Yigini

| Bilesen | Teknoloji |
|---------|-----------|
| Bulut | Google Cloud Platform (GCP) |
| Orkestrasyon | Google Kubernetes Engine (GKE) |
| Container Registry | Google Artifact Registry |
| CI/CD | GitHub Actions |
| IaC | Terraform |
| Monitoring | Prometheus |
| Reverse Proxy | Nginx |
| Veritabani | MongoDB Atlas |

## Mimari Diyagram

```
                              INTERNET
                                  |
                                  v
                    +----------------------------+
                    |     GCP Load Balancer       |
                    |       (External IP)         |
                    +----------------------------+
                                  |
                                  v
     +----------------------------------------------------------+
     |              Google Kubernetes Engine (GKE)               |
     |                  Namespace: mern-app                      |
     |                                                           |
     |  +-------------------+       +------------------------+  |
     |  | Frontend (Nginx)  |       |  Backend (Express.js)  |  |
     |  | - React static    | ----> |  - REST API            |  |
     |  | - Reverse proxy   |       |  - Port 5050           |  |
     |  | - 2 replica       |       |  - 2-5 replica (HPA)   |  |
     |  +-------------------+       +------------------------+  |
     |                                        |                  |
     |                                        v                  |
     |                              +------------------+         |
     |                              | MongoDB Atlas    |         |
     |                              | (Harici)         |         |
     |                              +------------------+         |
     |                                                           |
     |  +-------------------+       +------------------------+  |
     |  | Python ETL        |       | Prometheus             |  |
     |  | CronJob (saatlik) |       | Metrik + Alert         |  |
     |  +-------------------+       +------------------------+  |
     +----------------------------------------------------------+
```

## Bilesen Detaylari

### Frontend (React + Nginx)
- React SPA sunucu ve API isteklerini backend'e yonlendiren reverse proxy
- Multi-stage build: Node.js ile build, Nginx ile servis
- Port: 80, 2 replica
- Yonlendirme:
  - `/` -> React static dosyalar
  - `/record/*` -> Backend API (proxy)
  - `/healthcheck` -> Backend health (proxy)
  - `/nginx-health` -> Nginx kendi health check

### Backend (Express.js)
- CRUD islemleri icin REST API
- Port: 5050, 2-5 replica (HPA, CPU %70)
- Endpoint'ler:
  - `GET /record` - Kayitlari listele
  - `POST /record` - Kayit olustur
  - `GET /record/:id` - Tek kayit getir
  - `PATCH /record/:id` - Kayit guncelle
  - `DELETE /record/:id` - Kayit sil
  - `GET /healthcheck` - Saglik durumu

### MongoDB Atlas
- Yonetilen veritabani servisi (cluster icinde DB yok)
- Baglanti: Kubernetes Secret (ATLAS_URI)

### Python ETL
- GitHub API'den veri ceken ETL job'i
- Zamanlama: Her saat (`0 * * * *`)
- Kubernetes CronJob olarak calisir

### Prometheus
- Pod metriklerini 15 saniyede bir toplar
- Alert kurallari tanimli (CrashLoop, yuksek CPU, yuksek memory)
- Web UI: port 9090

## CI/CD Akisi

```
Gelistirici kod push eder
        |
        v
+-------------------+     +-------------------+     +-------------------------+
| CI Pipeline       |     |                   |     |                         |
| - Frontend test   | --> | Docker image      | --> | Artifact Registry'ye    |
| - Backend test    |     | build             |     | push                    |
+-------------------+     +-------------------+     +-------------------------+
                                                              |
                                                              v
                                                    +-------------------------+
                                                    | CD Pipeline             |
                                                    | - GCP auth              |
                                                    | - K8s manifest guncelle |
                                                    | - kubectl apply         |
                                                    | - Rollout dogrula       |
                                                    +-------------------------+
```

## Guvenlik Onlemleri

| Onlem | Uygulama |
|-------|----------|
| Non-root container | Tum Dockerfile'larda non-root user |
| Secret yonetimi | K8s Secrets ile MongoDB URI |
| Kaynak limitleri | Tum container'larda CPU/Memory limit |
| Guvenlik header'lari | X-Frame-Options, X-Content-Type-Options (Nginx) |
| RBAC | Minimum yetkili service account'lar |
| Network policy | Cluster seviyesinde aktif |

## Ag Akisi

1. Dis trafik GCP Load Balancer'a ulasir
2. Load Balancer, Frontend Service'e yonlendirir (port 80)
3. Nginx, UI istekleri icin React static dosyalarini sunar
4. Nginx, `/record` ve `/healthcheck` isteklerini Backend'e proxy yapar
5. Backend, TLS uzerinden MongoDB Atlas'a baglanir
6. Prometheus, pod metriklerini toplar ve alert kurallarini degerlendirir
