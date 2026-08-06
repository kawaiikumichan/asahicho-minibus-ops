# ASAHI Coach App - Deployment Architecture (物理・実行構成図)

本アーキテクチャ図は、ASAHI Coach App が実際にどのクラウドプラットフォーム上でどのように実行されているかを示す、物理・デプロイメント構成図です。CI/CD パイプラインからエッジネットワーク、および各サービスのホスティング環境を可視化しています。

> [!TIP]
> **SVG / PNG / draw.io 形式での出力方法**
> 本図は Mermaid.js 記法で作成されています。Draw.io の `[Arrange] -> [Insert] -> [Advanced] -> [Mermaid]` からインポートすることで、グラフィカルな図として保存・エクスポートが可能です。

```mermaid
flowchart TB
    classDef external fill:#eceff1,stroke:#607d8b,stroke-width:2px;
    classDef vercel fill:#000000,stroke:#ffffff,color:#ffffff,stroke-width:2px;
    classDef gcp fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef firebase fill:#fff8e1,stroke:#ff8f00,stroke-width:2px;
    classDef pipeline fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px;

    %% ユーザー環境
    subgraph Users ["User Environment"]
        Browser["Mobile / Desktop Browser (PWA)"]:::external
    end

    %% CI/CD
    subgraph CICD ["CI/CD Pipeline (GitHub)"]
        GitHub["GitHub Repository"]:::pipeline
        Actions["GitHub Actions (Test, Build, Deploy)"]:::pipeline
        GitHub --> Actions
    end

    %% ホスティング層 (Vercel)
    subgraph Hosting ["Vercel Edge Network / Serverless"]
        VercelCDN["Vercel Edge CDN"]:::vercel
        NextApp["Next.js App (SSR / Server Actions)"]:::vercel
        API["Next.js API Routes (Webhooks)"]:::vercel
        VercelCDN --> NextApp
        VercelCDN --> API
    end

    %% GCP / Firebase (Tokyo Region)
    subgraph CloudRegion ["Google Cloud - asia-northeast1 (Tokyo)"]
        
        subgraph Firebase ["Firebase Managed Services"]
            F_Auth["Firebase Authentication"]:::firebase
            F_DB[("Cloud Firestore<br/>(Multi-region / High Availability)")]:::firebase
            F_Storage["Cloud Storage"]:::firebase
        end

        subgraph GCPCore ["Google Cloud Services"]
            G_Secret["Secret Manager"]:::gcp
            G_Log["Cloud Logging & Error Reporting"]:::gcp
        end
    end

    %% 外部プロバイダ
    subgraph ExternalProviders ["External SaaS"]
        S_Stripe["Stripe (決済)"]:::external
        S_GMO["GMO Aozora (バーチャル口座)"]:::external
        S_Freee["freee (法定会計)"]:::external
    end

    %% ネットワーク接続
    Users -- HTTPS --> VercelCDN
    Actions -- "Automated Deploy" --> VercelCDN
    
    NextApp -- "SDK / RPC" --> F_DB
    NextApp -- "SDK / RPC" --> F_Auth
    API -- "Fetch Config" --> G_Secret
    API -- "Structured Logs" --> G_Log
    
    API -- "Server-to-Server" --> S_Freee
    S_Stripe -- "Webhook" --> API
    S_GMO -- "Webhook" --> API

```
