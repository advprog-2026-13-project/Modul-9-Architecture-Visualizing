# Visualisasi Arsitektur Yomu AdvPro-13
Repository ini merujuk pada aplikasi Yomu yang berada di organisasi AdvPro-13, arsitektur yang dipakai berupa Modular Monolith yang memiliki pola arsitektur Clean Architecture. Berikut adalah visualisasi dari arsitektur yang digunakan dalam projek ini.

## Context Diagram
![Context Diagram](images/context_diagram.drawio.png)
Context diagram menggambarkan Yomu sebagai satu sistem tunggal yang berinteraksi dengan dua aktor utama: Pelajar yang menggunakan aplikasi untuk membaca, mengerjakan kuis, dan berpartisipasi dalam sistem gamifikasi; serta Admin yang mengelola konten, moderasi, dan siklus liga. Sistem juga terhubung dengan layanan eksternal Google OAuth untuk keperluan autentikasi Single Sign-On.

## Container Diagram
![Container Diagram](images/container_diagram.drawio.png)
Container diagram menunjukkan bahwa Yomu terdiri dari tiga container utama: Next.js Frontend yang di-deploy di Vercel sebagai antarmuka pengguna, Spring Boot Monolith yang berjalan di AWS ECS Fargate sebagai inti pemrosesan logika bisnis, dan PostgreSQL sebagai penyimpanan data utama. Komunikasi antara frontend dan backend dilakukan melalui REST API over HTTPS, sementara komunikasi antar modul di dalam monolith menggunakan Spring ApplicationEvent untuk menjaga loose coupling.

## Deployment Diagram
![Deployment Diagram](images/deployment_diagram.drawio.png)
Deployment diagram menggambarkan infrastruktur Yomu saat ini yang berjalan pada satu environment: frontend di Vercel dan backend pada satu instance AWS ECS Fargate dengan satu database PostgreSQL. Arsitektur ini dipilih karena sederhana dan cukup untuk kebutuhan pengembangan awal tim kecil.

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
Berdasarkan hasil risk storming yang telah dilakukan, kami mengidentifikasi beberapa kerentanan utama pada arsitektur Yomu saat ini beserta mitigasi yang telah dan akan diterapkan.

### JWT di localStorage
Pada implementasi saat ini, JWT access token disimpan di localStorage browser. Ini merupakan kerentanan terhadap serangan Cross-Site Scripting (XSS). Apabila ada script berbahaya yang berhasil diinjeksikan ke halaman, token dapat dicuri dan digunakan untuk mengambil alih akun pengguna.
Mitigasi yang direncanakan: Memindahkan token ke httpOnly Cookie sehingga tidak dapat diakses oleh JavaScript sama sekali. Perubahan ini sudah tercermin pada arsitektur masa depan.

### Single PostgreSQL Instance (SPOF)
Arsitektur saat ini hanya menggunakan satu instance PostgreSQL tanpa failover. Jika database mengalami gangguan, seluruh platform akan ikut down karena semua modul bergantung pada satu sumber data yang sama.
Mitigasi yang direncanakan: Menggunakan RDS PostgreSQL Multi-AZ dengan Read Replica untuk memisahkan jalur read/write dan mengaktifkan auto-failover otomatis.

### Tidak Ada Cache Layer
Setiap request dari pengguna, termasuk untuk leaderboard dan data profil yang jarang berubah, langsung menyentuh database. Pada skala besar, ini akan menjadi bottleneck yang signifikan.
Mitigasi yang direncanakan: Menambahkan ElastiCache Redis sebagai cache layer. Data yang sering diakses seperti leaderboard dan session dapat di-cache sehingga mengurangi beban database secara drastis.

### Tidak Ada Rate Limiting
Tidak ada mekanisme pembatasan request saat ini, sehingga endpoint publik rentan terhadap serangan brute force maupun DDoS.
Mitigasi yang direncanakan: Menambahkan AWS WAF di edge layer dengan aturan rate limiting dan perlindungan OWASP untuk memblokir traffic berbahaya sebelum mencapai aplikasi.

## Individual Works

### Rifqi's Container
#### Container Diagram
```mermaid
graph TD
    Browser["Browser"]
    
    subgraph "Vercel"
        NextJS["Next.js 16 Frontend"]
        APIRoutes["Next.js API Routes<br/>BFF Proxy"]
    end
    
    subgraph "AWS ECS Fargate"
        LB["Application Load Balancer"]
        subgraph "Spring Boot Monolith"
            Auth["Auth Module"]
            Reading["Reading Module"]
            Achievements["Achievements Module"]
        end
    end
    
    subgraph "AWS"
        RDS["RDS PostgreSQL"]
        ElastiCache["ElastiCache Redis"]
    end
    
    Google["Google OAuth"]
    Browser -->|"HTTPS"| NextJS
    NextJS -->|"proxy /api/auth/*"| APIRoutes
    APIRoutes -->|"REST JSON"| LB
    LB --> Auth
    Auth -->|"JPA"| RDS
    Auth -->|"session cache"| ElastiCache
    Browser -->|"OAuth login"| Google
    Auth -->|"token verify"| Google
    Reading -.->|"event"| Achievements
```  

#### Component Diagram
```mermaid
graph TD
    subgraph "API Layer"
        ACL["AuthController<br/>POST /register<br/>POST /login<br/>POST /google"]
        MCL["MeController<br/>GET /me<br/>PATCH /me<br/>DELETE /me"]
    end
    subgraph "Application Layer"
        AS["AuthService<br/>register(RegisterRequest)<br/>login(LoginRequest)<br/>loginWithGoogle(String)<br/>updateUser(UUID, request)<br/>deleteUser(UUID)"]
        GS["GoogleService<br/>verifyToken(String)<br/>returns Payload"]
    end
    subgraph "Domain Layer"
        UE["User Entity<br/>id: UUID<br/>username: String<br/>displayName: String<br/>email: String<br/>phoneNumber: String<br/>passwordHash: String<br/>googleSub: String<br/>role: Role<br/>createdAt: Timestamp"]
        RE["Role Enum<br/>USER<br/>ADMIN"]
        DTO["DTOs<br/>RegisterRequest<br/>LoginRequest<br/>AuthResponse<br/>MeResponse<br/>UpdateAccountRequest"]
    end
    subgraph "Infrastructure Layer"
        UR["UserRepository<br/>findByUsername<br/>findByEmail<br/>findByPhoneNumber<br/>findByGoogleSub<br/>findById"]
        JWT["JwtService<br/>generateToken(User)<br/>parse(token): Payload"]
        JF["JwtAuthFilter<br/>doFilterInternal<br/>extract Bearer<br/>set SecurityContext"]
        SU["SecurityUser<br/>implements UserDetails<br/>getUser(): User"]
        SC["SecurityConfig<br/>stateless sessions<br/>BCrypt(12)<br/>permitAll: /auth/*<br/>authenticated: /**"]
    end
    subgraph "External"
        PGSQL["PostgreSQL"]
        GOOGLE["Google Identity"]
    end
    ACL --> AS
    MCL --> AS
    AS --> UR
    AS --> JWT
    AS --> GS
    AS --> DTO
    UR --> PGSQL
    GS --> GOOGLE
    JF --> JWT
    JF --> UR
    JF --> SU
    JF --> SC
```

#### Login, Register & Google SSO
```mermaid
sequenceDiagram
    actor User
    participant Client as Next.js Page
    participant GoogleUI as Google OAuth
    participant Proxy as Next.js API Route
    participant Ctrl as AuthController
    participant Svc as AuthService
    participant GS as GoogleService
    participant GID as Google Identity
    participant Repo as UserRepository
    participant BCrypt as BCryptPasswordEncoder
    participant JWT as JwtService
    participant DB as PostgreSQL
    rect rgb(230, 245, 255)
        Note over User,DB: REGISTRATION
        User->>Client: Fill register form<br/>{username, displayName, email, password}
        Client->>Proxy: POST /api/auth/register
        Proxy->>Ctrl: proxy to backend
        Ctrl->>Ctrl: @Valid RegisterRequest
        Ctrl->>Svc: register(request)
        Svc->>Repo: findByUsername(username)
        Repo->>DB: SELECT WHERE username = ?
        DB-->>Repo: null
        Svc->>Repo: findByEmail(email)
        DB-->>Repo: null
        Svc->>BCrypt: encode(password)
        BCrypt-->>Svc: passwordHash
        Svc->>Svc: new User(role=USER, now)
        Svc->>Repo: save(user)
        Repo->>DB: INSERT INTO users
        DB-->>Repo: user with UUID
        Svc->>JWT: generateToken(user)
        JWT-->>Svc: accessToken
        Svc-->>Ctrl: AuthResponse(accessToken)
        Ctrl-->>Proxy: 200 {accessToken}
        Proxy-->>Client: 200 {accessToken}
    end
    rect rgb(255, 248, 230)
        Note over User,DB: LOGIN
        User->>Client: Fill login form<br/>{identifier, password}
        Client->>Proxy: POST /api/auth/login
        Proxy->>Ctrl: proxy to backend
        Ctrl->>Ctrl: @Valid LoginRequest
        Ctrl->>Svc: login(request)
        Svc->>Repo: findByUsername(identifier)
        Repo->>DB: SELECT WHERE username = ?
        DB-->>Repo: null
        Svc->>Repo: findByEmail(identifier)
        Repo->>DB: SELECT WHERE email = ?
        DB-->>Repo: user
        Svc->>BCrypt: matches(password, user.passwordHash)
        BCrypt-->>Svc: true
        Svc->>JWT: generateToken(user)
        JWT-->>Svc: accessToken
        Svc-->>Ctrl: AuthResponse(accessToken)
        Ctrl-->>Proxy: 200 {accessToken}
        Proxy-->>Client: 200 {accessToken}
        Client->>Client: localStorage.setItem("token", accessToken)
        Client-->>User: Redirect to dashboard
    end
    rect rgb(235, 255, 235)
        Note over User,DB: GOOGLE SSO
        User->>Client: Click "Sign in with Google"
        Client->>GoogleUI: Open Google OAuth popup
        GoogleUI->>User: Select Google account
        GoogleUI-->>Client: Google ID Token
        Client->>Proxy: POST /api/auth/google<br/>{token: idTokenString}
        Proxy->>Ctrl: proxy to backend
        Ctrl->>Svc: loginWithGoogle(idTokenString)
        Svc->>GS: verifyToken(idTokenString)
        GS->>GS: Build GoogleIdTokenVerifier(clientId)
        GS->>GID: verifier.verify(idTokenString)
        GID-->>GS: GoogleIdToken
        GS-->>Svc: Payload(email, sub, name)
        Svc->>Repo: findByGoogleSub(sub)
        Repo->>DB: SELECT WHERE google_sub = ?
        DB-->>Repo: null
        Svc->>Repo: findByEmail(email)
        Repo->>DB: SELECT WHERE email = ?
        DB-->>Repo: null
        Svc->>Svc: new User(googleSub=sub, email, name,<br/>role=USER, now)
        Svc->>Repo: save(user)
        Repo->>DB: INSERT INTO users
        DB-->>Repo: saved
        Svc->>JWT: generateToken(user)
        JWT-->>Svc: accessToken
        Svc-->>Ctrl: AuthResponse(accessToken)
        Ctrl-->>Proxy: 200 {accessToken}
        Proxy-->>Client: 200 {accessToken}
        Client-->>User: Redirect to dashboard
    end
    Note over Repo,DB: All three flows end with JWT generation<br/>and return accessToken to client
```


#### JWT Filter & Authenticated Request
```mermaid
sequenceDiagram
    actor User
    participant Client as Next.js Page
    participant Proxy as Next.js API Route
    participant Filter as JwtAuthFilter
    participant JWT as JwtService
    participant Repo as UserRepository
    participant SC as SecurityContext
    participant Ctrl as MeController
    participant Svc as AuthService
    participant DB as PostgreSQL
    User->>Client: Visit dashboard
    Client->>Client: token = localStorage.getItem("token")
    Client->>Proxy: GET /api/auth/me<br/>Authorization: Bearer {token}
    Proxy->>Proxy: Check Authorization header
    alt No header
        Proxy-->>Client: 401 Unauthorized
    end
    Proxy->>Filter: forward to backend<br/>Authorization: Bearer {token}
    Filter->>Filter: Extract header
    alt No "Bearer " prefix
        Filter->>Filter: chain.doFilter() — skip
    end
    Filter->>JWT: parse(token)
    JWT->>JWT: verify HMAC signature<br/>extract claims
    JWT-->>Filter: Payload(userId, username, role)
    Filter->>Repo: findById(UUID.fromString(userId))
    Repo->>DB: SELECT * FROM users WHERE id = ?
    DB-->>Repo: user row
    Repo-->>Filter: User entity
    Filter->>Filter: new SecurityUser(user)
    Filter->>SC: setAuthentication(<br/>  UsernamePasswordAuthenticationToken(<br/>    SecurityUser, null, [ROLE_USER]))
    Filter->>Filter: chain.doFilter()
    Ctrl->>SC: getAuthentication().getPrincipal()
    SC-->>Ctrl: SecurityUser
    Ctrl->>Ctrl: user = securityUser.getUser()
    Ctrl->>Svc: getCurrentUser(user.getId())
    Svc->>Repo: findById(userId)
    Repo->>DB: SELECT * FROM users WHERE id = ?
    DB-->>Repo: user
    Svc-->>Ctrl: User
    Ctrl-->>Proxy: MeResponse(user)
    Proxy-->>Client: 200 {id, username, email, role...}
    Client-->>User: Render dashboard profile
    Note over JWT,Filter: If token expired/invalid<br/>catch → log error → clearContext → 401
```

### Nadya's Container

### Azzaka's Container
##### Container Diagram
![Achievement Container Diagram](images/container_diagram_azzaka.drawio.png)

##### Code Diagram — Strategy Pattern
![Azzaka Code Diagram Strategy](images/code_diagram_azzaka_strategy.drawio.png)

##### Code Diagram — Entities
![Azzaka Code Diagram Entities](images/code_diagram_azzaka_entities.drawio.png)

### Marco's Container
##### Container Diagram
![Forum Container Diagram](images/container_diagram_marco.drawio.png)

##### Component Diagram
![Forum Component Diagram](images/component_diagram_marco.drawio.png)

### Muhathir's Container
##### Container Diagram
![Achievement Container Diagram](images/container_diagram_muhathir.drawio.png)

##### Code Diagram
![Achievement Code Diagram](images/code_diagram_muhathir.drawio.png)