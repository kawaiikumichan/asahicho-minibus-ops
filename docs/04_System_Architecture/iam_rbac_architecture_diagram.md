# ASAHI Coach App - IAM & RBAC Architecture Diagram

本図は、ASAHI Coach App における Identity & Access Management（IAM）およびロールベースアクセス制御（RBAC）の境界を示すアーキテクチャ図です。
各ロール（アクター）が「誰が」「何を」「どこまで」操作可能か（ADR-004）を可視化しており、ISO27001やSOC2等のセキュリティ監査に対応するための権限境界を証明します。

> [!TIP]
> **SVG / PNG / draw.io 形式での出力方法**
> 本図は Mermaid.js 記法で作成されています。Draw.io の `[Arrange] -> [Insert] -> [Advanced] -> [Mermaid]` からインポートすることで、グラフィカルな図として保存・エクスポートが可能です。

```mermaid
flowchart LR
    classDef actor fill:#e8eaf6,stroke:#3f51b5,stroke-width:2px;
    classDef resource fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,shape:rect;
    classDef noaccess fill:#ffebee,stroke:#b71c1c,stroke-width:2px,stroke-dasharray: 5 5;
    classDef system fill:#fff3e0,stroke:#e65100,stroke-width:2px;

    Parent(("Parent<br/>(保護者)")):::actor
    Coach(("Coach<br/>(指導者)")):::actor
    Treasurer(("Treasurer<br/>(会計・経理)")):::actor
    Admin(("Admin<br/>(システム管理者)")):::actor
    System(("System<br/>(自動バッチ/Webhook)")):::system

    %% 保護者の権限
    subgraph Parent_Access ["Parent Permissions (User)"]
        direction TB
        P_Att[Attendance<br/>(出欠登録・閲覧)]:::resource
        P_Inv[Invoice<br/>(自身の請求閲覧)]:::resource
        P_Wal[Wallet<br/>(自身の残高閲覧)]:::resource
    end
    Parent -->|"参照・限定更新"| Parent_Access

    %% 指導者の権限
    subgraph Coach_Access ["Coach Permissions (Manager)"]
        direction TB
        C_Ride[Ride<br/>(配車作成・クレジット承認)]:::resource
        C_Att[Attendance<br/>(全体の出欠閲覧)]:::resource
        C_Man[Manual<br/>(運用マニュアル作成)]:::resource
    end
    Coach -->|"業務運用・承認"| Coach_Access

    %% 会計担当の権限
    subgraph Treasurer_Access ["Treasurer Permissions (Finance)"]
        direction TB
        T_Inv[Invoice<br/>(請求確定・手動相殺)]:::resource
        T_Jour[Journal<br/>(仕訳データ閲覧・編集)]:::resource
        T_Freee[freee Export<br/>(法定会計への連携)]:::resource
    end
    Treasurer -->|"財務管理・決済"| Treasurer_Access

    %% 管理者の権限
    subgraph Admin_Access ["Admin Permissions (Superuser)"]
        direction TB
        A_User[User Management<br/>(アカウント管理)]:::resource
        A_RBAC[RBAC Control<br/>(権限の付与・剥奪)]:::resource
        A_Sec["Secrets (Env Vars)<br/>(❌ 閲覧不可・アクセス拒否)"]:::noaccess
        A_Audit[Audit Log<br/>(システム監査ログ閲覧)]:::resource
    end
    Admin -->|"システム管理"| Admin_Access

    %% システムの権限 (Service Account)
    subgraph System_Access ["System Services (Internal API)"]
        direction TB
        S_Jour[Journal<br/>(自動仕訳生成)]:::resource
        S_Pay[Payment<br/>(決済ステータス自動反映)]:::resource
        S_Sec[Secrets<br/>(APIキーのセキュア取得)]:::resource
        S_Log[Cloud Logging<br/>(エラー・監査ログの永続化)]:::resource
    end
    System -->|"自動化バッチ実行"| System_Access

```
