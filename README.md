# MERN Stack DevOps Case Study

MERN stack uygulamasi ve Python ETL projesinin GCP (GKE) uzerine deploy edilmesi. CI/CD, monitoring ve IaC dahil.

**Canli Uygulama:** http://35.240.117.248

## Teknoloji Yigini

| Bilesen | Teknoloji |
|---------|-----------|
| Frontend | React + Nginx (reverse proxy) |
| Backend | Express.js + Node.js |
| Veritabani | MongoDB Atlas |
| Python ETL | Python 3.11 (CronJob, saatlik) |
| Bulut | Google Cloud Platform (GKE) |
| CI/CD | GitHub Actions |
| IaC | Terraform |
| Monitoring | Prometheus |

## Proje Yapisi

```
.
├── .github/workflows/
│   ├── ci.yml                  # CI: test, build, push
│   └── cd.yml                  # CD: deploy to GKE
├── mern-project/
│   ├── client/                 # React frontend
│   │   ├── Dockerfile
│   │   ├── nginx.conf
│   │   └── src/
│   └── server/                 # Express.js backend
│       ├── Dockerfile
│       └── routes/
├── python-project/             # Python ETL
│   ├── Dockerfile
│   ├── requirements.txt
│   └── ETL.py
├── k8s/                        # Kubernetes manifest'leri
│   ├── namespace.yaml
│   ├── mongodb/                # Atlas secret
│   ├── backend/                # Deployment, Service, HPA
│   ├── frontend/               # Deployment, Service, ConfigMap
│   ├── python-etl/             # CronJob
│   └── monitoring/             # Prometheus
├── terraform/                  # Altyapi kodu
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── provider.tf
├── docs/
│   ├── architecture.md
│   └── deployment-guide.md
├── docker-compose.yml
└── README.md
```

## Hizli Baslangic

### Gereksinimler

- GCP hesabi (faturalandirma aktif)
- `gcloud`, `terraform`, `kubectl`, `docker` kurulu
- MongoDB Atlas cluster'i hazir



## Mimari

Detaylar icin: [Architecture](docs/architecture.md)

## Ozellikler

- **Konteynerlestime**: Multi-stage Docker build, non-root user
- **Orkestrasyon**: Kubernetes, HPA ile otomatik olcekleme, rolling update, health check
- **CI/CD**: GitHub Actions ile otomatik build, test, deploy
- **Monitoring**: Prometheus ile metrik toplama ve alert kurallari
- **IaC**: Terraform ile GKE cluster ve Artifact Registry yonetimi
- **Guvenlik**: Non-root container, K8s Secrets, RBAC, guvenlik header'lari

## Karsilasilan Zorluklar ve Cozumler

### 1. Docker Image Mimari Uyumsuzlugu

**Sorun:** Apple Silicon (ARM64) makinede build edilen image'lar GKE'de `exec format error` ile crash oldu.

**Sebep:** GKE node'lari AMD64 mimarisi kullaniyor, lokal makinede ARM64 image uretilmisti.

**Cozum:** `--platform linux/amd64` flagi ile yeniden build edildi:
```bash
docker build --platform linux/amd64 -t IMAGE:TAG ./path
```

### 2. MongoDB Atlas Baglanti Hatasi

**Sorun:** Backend pod'lari `tlsv1 alert internal error` SSL hatasi ile crash oldu.

**Sebep:** MongoDB Atlas Network Access ayarlarinda GKE node IP'lerine izin verilmemisti.

**Cozum:** Atlas > Network Access > `0.0.0.0/0` eklendi. 

**Not:** Production ortamında `0.0.0.0/0` güvenlik riski oluşturur. İdeal çözüm GKE node'larının çıkış IP'lerini (NAT Gateway üzerinden sabit IP) Atlas whitelist'ine eklemektir. Ancak bu case study kapsamında:
- Cloud NAT ek maliyet ve kompleksite getirir
- Atlas zaten TLS encryption ve authentication zorunlu tutar
- Veritabanı erişimi güçlü şifre + connection string ile korunur

Bu nedenle case study ortamı için kabul edilebilir bir trade-off olarak değerlendirilmiştir.

### 3. Yetersiz CPU Kaynaklari

**Sorun:** Monitoring pod'lari `Pending` durumunda kaldi -- node'larda `Insufficient cpu`.

**Sebep:** e2-medium node'larda GKE sistem pod'lari CPU request'lerinin buyuk kismini kullaniyordu.

**Cozum:** Monitoring stack sadece Prometheus'a indirildi, resource request'ler minimize edildi.

### 4. Frontend Hardcoded URL'ler

**Sorun:** React bilesenleri `http://localhost:5050` adresine istek yapiyordu. Production'da calismaz.

**Cozum:** Tum URL'ler relative path'e cevrildi, Nginx reverse proxy backend'e yonlendiriyor:
```javascript
// Onceki: fetch("http://localhost:5050/record")
// Sonraki:
fetch("/record")
```


