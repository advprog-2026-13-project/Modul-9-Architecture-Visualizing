# Visualisasi Arsitektur Yomu AdvPro-13
Repository ini merujuk pada aplikasi Yomu yang berada di organisasi AdvPro-13, arsitektur yang dipakai berupa Modular Monolith yang memiliki pola arsitektur Clean Architecture. Berikut adalah visualisasi dari arsitektur yang digunakan dalam projek ini.

## Context Diagram
![Context Diagram](images/context_diagram.drawio.png)

## Container Diagram
![Container Diagram](images/container_diagram.drawio.png)

## Deployment Diagram
![Deployment Diagram](images/deployment_diagram.drawio.png)

## Rencana Arsitektur Aplikasi di Masa Depan
```mermaid
graph TD
    subgraph "Edge"
        CF["CloudFront CDN<br/>Static assets + HTTPS"]
        WAF["AWS WAF<br/>Rate limiting + OWASP rules"]
    end

    subgraph "Vercel"
        FE["Next.js Frontend<br/>SSR + Static"]
    end

    subgraph "AWS VPC"
        ALB["Application Load Balancer<br/>Public subnet"]

        subgraph "ECS Fargate — Private Subnet"
            ECS1["Spring Boot<br/>Task 1"]
            ECS2["Spring Boot<br/>Task 2"]
            ECS3["Spring Boot<br/>Task N"]
        end

        subgraph "Caching"
            ELC["ElastiCache Redis<br/>Session + Hot data<br/>Rate limiter backend"]
        end

        subgraph "Database"
            RDS_M["RDS PostgreSQL<br/>Multi-AZ Master"]
            RDS_R["RDS PostgreSQL<br/>Read Replica"]
        end
    end

    subgraph "Monitoring"
        CW["CloudWatch<br/>Logs + Metrics + Alarms"]
        Grafana["Grafana<br/>Dashboards"]
    end

    subgraph "External"
        Google["Google OAuth"]
    end

    CF --> WAF
    WAF --> FE
    WAF --> ALB

    FE --> ALB

    ALB --> ECS1
    ALB --> ECS2
    ALB --> ECS3

    ECS1 --> ELC
    ECS2 --> ELC
    ECS3 --> ELC

    ECS1 --> RDS_M
    ECS2 --> RDS_M
    ECS3 --> RDS_M

    RDS_M -.->|"Replication"| RDS_R

    ECS1 --> RDS_R
    ECS2 --> RDS_R
    ECS3 --> RDS_R

    ECS1 --> CW
    ECS2 --> CW

    CW --> Grafana

    ECS1 --> Google
    FE --> Google
```
## Bagaimana kami menerapkan risk storming

Risk Storming adalah teknik kolaboratif untuk mengidentifikasi risiko arsitektur dengan memvisualisasikan skenario kegagalan sebelum terjadi. Kami menerapkannya pada arsitektur Yomu dengan pertanyaan: Jika proyek ini sukses dan mencapai 100K+ pengguna aktif harian besok, apa yang pertama kali akan rusak?

### Cara Kami Mengidentifikasi Risiko

Untuk setiap lapisan arsitektur, kami mengajukan tiga pertanyaan:

| Pertanyaan | Contoh |
|---|---|
| Apa yang terjadi jika ini gagal? | PostgreSQL crash → seluruh platform down (SPOF) |
| Apa yang terjadi jika ini kewalahan? | 1000 submission quiz bersamaan → single instance kolaps |
| Apa yang terjadi jika ini diserang? | XSS mencuri JWT dari localStorage → pengambilalihan akun massal |

Kami kemudian memetakan setiap risiko ke lapisan arsitekturnya dan menentukan tingkat keparahan:

| Layer | Risiko yang Ditemukan |
|---|---|
| Security | JWT di localStorage — permukaan serangan XSS |
| Data | Single PostgreSQL instance — tanpa failover, tanpa read scaling |
| Compute | Single compute instance — tanpa horizontal scaling |
| Caching | Tidak ada cache layer — setiap request langsung ke DB |
| Observability | Tidak ada monitoring — buta terhadap gangguan |
| API Protection | Tidak ada rate limiting — rentan DDoS |

---

### Bagaimana Risiko Membentuk Arsitektur Masa Depan

Setiap mitigasi secara langsung mempengaruhi desain arsitektur:

| Risiko | Mitigasi | Perubahan Arsitektur |
|---|---|---|
| JWT terekspos ke JavaScript | httpOnly Cookie | Token dipindahkan ke secure cookie — vektor XSS dihilangkan |
| Database single point of failure | Multi-AZ RDS + Read Replica | Jalur read/write dipisahkan, auto-failover |
| Tidak bisa scaling melebihi satu instance | ECS Fargate auto-scale | Kapasitas elastis yang menyesuaikan permintaan |
| Setiap request langsung ke DB | ElastiCache Redis | 80%+ traffic read diserap di cache layer |
| Tidak ada visibilitas ke kesehatan sistem | CloudWatch + Grafana | Dashboard real-time, alert anomali |
| Tidak ada proteksi penyalahgunaan | AWS WAF + rate limiting | Memblokir traffic berbahaya di edge |

---

### Mengapa kami memilih untuk Tetap Monolith

Risk storming juga memvalidasi keputusan kami untuk tidak memecah sistem menjadi microservices. Risiko yang kami temukan, single database, single instance, tidak ada caching, adalah masalah infrastruktur, bukan masalah arsitektur. Memecah menjadi microservices justru akan menambah lebih banyak risiko (network latency, distributed tracing, konsistensi data) tanpa menyelesaikan akar masalahnya. Modular monolith di atas infrastruktur yang scalable (ECS, RDS, ElastiCache) mengatasi risiko nyata sambil menjaga kompleksitas operasional tetap manageable untuk tim kecil.
## Kerentanan dan Keamanan Aplikasi
...

## Individual Works

### Rifqi's Container

### Nadya's Container

### Azzaka's Container

### Marco's Container

### Muhathir's Container
##### Container Diagram
![Achievement Container Diagram](images/container_diagram_muhathir.drawio.png)

##### Code Diagram
![Achievement Code Diagram](images/code_diagram_muhathir.drawio.png)