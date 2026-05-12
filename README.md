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