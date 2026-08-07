# ASAHI Coach App - Payment & Accounting Sequence Diagram

本図は、外部決済プロバイダ（Stripe 等）からの Webhook 受信を起点とした、「決済の消込」「複式簿記仕訳の生成」「監査ログへの記録」「法定会計（freee）へのエクスポート」に至る一連の時系列データ処理フローを示すシーケンス図です。
IR-001（インシデントレスポンス）や IR-002 等における、トランザクションの追跡性（Correlation ID）と処理の完全性を証明するための監査資料として利用します。

> [!TIP]
> **SVG / PNG / draw.io 形式での出力方法**
> 本図は Mermaid.js 記法で作成されています。Draw.io の `[Arrange] -> [Insert] -> [Advanced] -> [Mermaid]` からインポートすることで、グラフィカルな図として保存・エクスポートが可能です。

```mermaid
sequenceDiagram
    autonumber
    
    %% Actors & Participants
    actor Stripe as Stripe<br/>(決済基盤)
    participant API as API Route<br/>(Webhook Handler)
    participant RBAC as Policy Engine<br/>(RBAC)
    participant Payment as PaymentRecord<br/>(Firestore)
    participant Journal as JournalEntry<br/>(Firestore)
    participant Audit as Cloud Logging<br/>(監査ログ)
    actor Freee as freee API<br/>(法定会計)

    %% Flow Sequence
    Note over Stripe, API: [フェーズ1] 決済完了の非同期通知
    Stripe->>API: Webhook Request (決済完了イベント)
    activate API
    
    API->>API: 署名確認 (Verify Stripe Signature)
    
    Note over API, RBAC: [フェーズ2] セキュリティと権限の検証
    API->>RBAC: アクセス権限確認 (Role: System)
    activate RBAC
    RBAC-->>API: 許可 (Authorized)
    deactivate RBAC

    Note over API, Payment: [フェーズ3] 決済データの消込
    API->>Payment: 該当 Invoice を特定し消込 (Status: Paid)
    activate Payment
    Payment-->>API: 消込完了 (OK)
    deactivate Payment

    Note over API, Journal: [フェーズ4] 会計処理と証跡の記録
    API->>Journal: 複式簿記ルールに基づく仕訳生成 (借方/貸方)
    activate Journal
    Journal-->>API: 仕訳データ作成 (OK)
    deactivate Journal

    API->>Audit: トランザクション証跡記録 (with Correlation ID)
    activate Audit
    Audit-->>API: ログ記録完了
    deactivate Audit

    Note over API, Freee: [フェーズ5] 外部システム（法定会計）連携
    API->>Freee: 仕訳データ連携 (Deal / Journal Export)
    activate Freee
    Freee-->>API: 同期完了 (200 OK)
    deactivate Freee

    Note over API, Stripe: [完了] Webhook の正常終了応答
    API-->>Stripe: Ack (200 OK)
    deactivate API

```
