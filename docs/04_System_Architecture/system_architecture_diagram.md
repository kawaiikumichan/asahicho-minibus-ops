# ASAHI Coach App - System Architecture Diagram (Production Readiness Review)

本アーキテクチャ図は、ASAHI Coach App の本番運用審査（Production Readiness Review）に向けた公式システム構成図です。  
クライアント、アプリケーション、ドメイン境界、インフラストラクチャ、およびセキュリティバウンダリ（Trust Boundary）の全体像を網羅しています。

> [!TIP]
> **SVG / PNG / draw.io 形式での出力方法**
> 本図は Mermaid.js 記法で作成されています。以下の手順で要求された全フォーマットへの変換・保存が可能です。
> 1. **Draw.io へのインポート**: Draw.io (`app.diagrams.net`) を開き、メニューから `[Arrange] -> [Insert] -> [Advanced] -> [Mermaid]` を選択し、以下のコードブロックを貼り付けて [Insert] を押します。
> 2. **ファイル保存**: そのまま `.drawio` 形式で保存できます。
> 3. **画像エクスポート**: メニューの `[File] -> [Export as]` から `SVG` または `PNG` 形式を選択して出力してください。

```mermaid
flowchart TB
    %% Styling Definitions
    classDef clientLayer fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef appLayer fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef domainLayer fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef infraLayer fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;
    classDef externalLayer fill:#ffebee,stroke:#b71c1c,stroke-width:2px;
    classDef securityLayer fill:#eceff1,stroke:#37474f,stroke-width:2px,stroke-dasharray: 5 5;
    classDef obsLayer fill:#fffde7,stroke:#f57f17,stroke-width:2px;
    classDef drLayer fill:#e0f7fa,stroke:#006064,stroke-width:2px;
    classDef runbook fill:#263238,stroke:#ffffff,color:#ffffff,stroke-width:1px;

    %% ==========================================
    %% 1. Trust Boundary: Internet / Client Layer
    %% ==========================================
    subgraph Boundary_Internet [🌐 Trust Boundary: Internet / Client Layer]
        direction LR
        Client_Parent(Parent App):::clientLayer
        Client_Coach(Coach App):::clientLayer
        Client_Admin(Admin Portal):::clientLayer
    end

    %% ==========================================
    %% 2. Trust Boundary: Application Layer
    %% ==========================================
    subgraph Boundary_Application [🖥️ Trust Boundary: Application Layer - Next.js]
        direction TB
        App_UI[UI Components]:::appLayer
        App_API[API Routes / Server Actions]:::appLayer
        App_AuthN[Authentication Middleware]:::appLayer
        App_RBAC{"RBAC Policy Engine<br/><br/>ADR-004<br/>Fail Closed / Pure Function"}:::securityLayer
        App_DomainSvc[Domain Services]:::appLayer
        
        App_UI -->|Internal Call| App_API
        App_API --> App_AuthN
        App_AuthN --> App_RBAC
        App_RBAC -->|Authorized| App_DomainSvc
    end

    %% HTTPS/TLS Connection
    Client_Parent == HTTPS / TLS ==> App_UI
    Client_Coach == HTTPS / TLS ==> App_UI
    Client_Admin == HTTPS / TLS ==> App_UI

    %% ==========================================
    %% 3. Trust Boundary: Domain Layer (DDD)
    %% ==========================================
    subgraph Boundary_Domain [🏗️ Domain Layer - DDD Aggregates & Accounting Flow]
        direction TB
        
        %% Core Entities
        Dom_User((User)):::domainLayer
        Dom_Family((Family)):::domainLayer
        
        %% Accounting / Lifecycle Flow
        Dom_Attendance((Attendance)):::domainLayer
        Dom_Ride((Ride)):::domainLayer
        Dom_Wallet((WalletLedger)):::domainLayer
        Dom_Invoice((Invoice)):::domainLayer
        Dom_Payment((PaymentRecord)):::domainLayer
        Dom_Journal((JournalEntry)):::domainLayer

        %% Relationships (One-way dependencies)
        Dom_User -.->|Belongs to| Dom_Family
        Dom_Family -.->|Creates| Dom_Attendance
        
        %% Accounting Flow (Data Flow)
        Dom_Attendance ==>|Phase 8| Dom_Ride
        Dom_Ride ==>|Grant Credit| Dom_Wallet
        Dom_Wallet ==>|Offset| Dom_Invoice
        Dom_Invoice ==>|Paid| Dom_Payment
        Dom_Payment ==>|Bookkeeping| Dom_Journal
        
        %% ADR References Note
        Note_ADR["ADR Compliance:<br/>ADR-001, ADR-002, ADR-003<br/>ADR-010, ADR-011"]:::runbook
    end
    
    App_DomainSvc --> Boundary_Domain

    %% ==========================================
    %% 4. Trust Boundary: Cloud Infrastructure
    %% ==========================================
    subgraph Boundary_Cloud [☁️ Trust Boundary: Cloud Infrastructure & Observability]
        direction LR
        
        subgraph GCP_Core [Google Cloud Platform]
            direction TB
            SecretMgr["Google Secret Manager<br/>⚠️ Only layer holding Secrets"]:::securityLayer
            
            %% Observability (RSK-001)
            subgraph GCP_Observability [Observability & Monitoring]
                direction TB
                Obs_Logger["CloudLogger + AsyncLocalStorage<br/>(Correlation ID Context)"]:::obsLayer
                Obs_Logging[Cloud Logging]:::obsLayer
                Obs_Error[Cloud Error Reporting]:::obsLayer
                Obs_Monitor[Cloud Monitoring]:::obsLayer
                
                Obs_Logger --> Obs_Logging
                Obs_Logging --> Obs_Error
                Obs_Logging --> Obs_Monitor
            end
        end

        subgraph Firebase_Core [Firebase Backend]
            direction TB
            FB_Auth[Authentication]:::infraLayer
            FB_Firestore[(Firestore)]:::infraLayer
            FB_Storage[Cloud Storage]:::infraLayer
        end

        %% Disaster Recovery (RSK-004 / DR-002)
        subgraph DR_Flow [Disaster Recovery Flow - DR-002]
            direction LR
            DR1[(Firestore)]:::drLayer -->|PITR Restore| DR2(Temporary Project):::drLayer
            DR2 -->|Diff Verification| DR3{Approval}:::drLayer
            DR3 -->|Approved| DR4(Merge to Prod):::drLayer
        end
    end

    %% Infrastructure Links
    App_API -.->|Fetch Keys| SecretMgr
    App_DomainSvc -->|Mutate Data| FB_Firestore
    FB_Firestore -.-> DR1

    %% ==========================================
    %% 5. Trust Boundary: External Providers
    %% ==========================================
    subgraph Boundary_External [🌍 Trust Boundary: External Providers]
        direction LR
        Ext_Stripe[Stripe]:::externalLayer
        Ext_GMO[GMO Aozora]:::externalLayer
        Ext_Freee[freee Accounting]:::externalLayer
    end

    %% External Connections & Correlation Trace (RSK-001)
    Ext_Stripe == Webhook + X-Correlation-ID ==> App_API
    Ext_GMO == Webhook + X-Correlation-ID ==> App_API
    App_API -.->|Inject Async Context| Obs_Logger
    
    %% Accounting Flow to External (ADR-015)
    Dom_Journal == Export Deal ==> Ext_Freee

    %% Explicit Correlation ID Trace Representation
    Ext_Stripe ~~~|RSK-001 Trace Flow| Obs_Logging
    Dom_Payment ~~~|RSK-001 Trace Flow| Obs_Logging
    Dom_Journal ~~~|RSK-001 Trace Flow| Obs_Logging

    %% ==========================================
    %% 6. Runbook Mapping Annotation
    %% ==========================================
    RunbookNote>Mapped Runbooks: IR-001, IR-002, MO-001, MO-002, DR-001, DR-002, RM-001]:::runbook
```
