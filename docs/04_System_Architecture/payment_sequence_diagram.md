# ASAHI Coach App - Payment & Accounting Sequence Diagram

本図は、外部決済プロバイダ（Stripe 等）からの Webhook 受信を起点とした、「決済の消込」「複式簿記仕訳の生成」「監査ログへの記録」「法定会計（freee）へのエクスポート」に至る一連の時系列データ処理フローを示すシーケンス図です。
IR-001（インシデントレスポンス）や IR-002 等における、トランザクションの追跡性（Correlation ID）と処理の完全性を証明するための監査資料として利用します。

本図は正常系に加え、**異常系（署名検証失敗・重複受信・耐久化失敗・エクスポート失敗）の分岐と応答コードを明示**します。応答コードの決定規則は ADR-022（エラー伝播・失敗可視化原則）2.3 Ack 境界原則に従い、`freee` への法定会計エクスポートは ADR-010 / ADR-015 に基づき Webhook 同期区間から切り離した非同期ワーカーが担当します。

> [!TIP]
> **SVG / PNG / draw.io 形式での出力方法**
> 本図は Mermaid.js 記法で作成されています。Draw.io の `[Arrange] -> [Insert] -> [Advanced] -> [Mermaid]` からインポートすることで、グラフィカルな図として保存・エクスポートが可能です。

```mermaid
sequenceDiagram
    autonumber

    %% Actors & Participants
    actor Stripe as Stripe<br/>(決済基盤)
    participant API as API Route<br/>(Webhook Handler)
    participant Evidence as PaymentEvidence<br/>(Firestore)
    participant RBAC as Policy Engine<br/>(RBAC)
    participant Payment as PaymentRecord / Invoice<br/>(Firestore)
    participant Journal as JournalEntry + Outbox<br/>(Firestore)
    participant Audit as Cloud Logging<br/>(監査ログ)
    participant Worker as Export Worker<br/>(非同期)
    participant Alert as Cloud Monitoring<br/>(アラート)
    actor Freee as freee API<br/>(法定会計)

    %% Flow Sequence
    Note over Stripe, API: [フェーズ1] 決済完了の非同期通知
    Stripe->>API: Webhook Request (決済完了イベント)
    activate API

    API->>API: 署名確認 (Verify Stripe Signature)

    alt 署名検証 NG (failureClass: SECURITY)
        API->>Evidence: signatureVerificationResult=FAILED / rawEventHash 保存
        API->>Alert: 検証失敗アラート (しきい値超過時 / IR-001)
        API-->>Stripe: 400 Bad Request (再送させない)
    else 署名検証 OK
        Note over API, Payment: [フェーズ2] べき等性判定 (ADR-020 / Rule-WAL-004)
        API->>Evidence: providerEventId の既処理チェック
        alt 既に処理済 (重複再送)
            API->>Audit: 重複受信を記録 (No-Op)
            API-->>Stripe: 200 OK (No-Op)
        else 未処理
            Note over API, RBAC: [フェーズ3] 権限の検証
            API->>RBAC: アクセス権限確認 (Role: System)
            activate RBAC
            RBAC-->>API: 許可 / Fail-Closed DENY (failureClass 付与)
            deactivate RBAC

            Note over API, Journal: [フェーズ4] 消込・仕訳・Outbox を単一トランザクションで耐久化
            API->>Payment: Invoice 消込 (Status: PAID)
            API->>Journal: 複式簿記仕訳生成＋Export Outbox イベント登録
            alt トランザクション失敗 (failureClass: RETRYABLE_TRANSIENT / CONFLICT)
                API->>Audit: severity=ERROR ログ (correlationId, failureClass, cause)
                API-->>Stripe: 5xx (プロバイダ再送に委ねる)
            else コミット成功
                API->>Audit: トランザクション証跡記録 (with Correlation ID)
                API->>Evidence: signatureVerificationResult=VERIFIED / processedAt 保存
                Note over API, Stripe: 耐久化完了後にのみ Ack (ADR-022 2.3)
                API-->>Stripe: Ack (200 OK)
            end
        end
    end
    deactivate API

    Note over Worker, Freee: [フェーズ5] 法定会計エクスポート (Webhook 同期区間から分離 / ADR-015)
    Worker->>Journal: 未送信 JournalEntry / Outbox イベント取得
    activate Worker
    Worker->>Freee: 仕訳データ連携 (Deal / Journal Export)
    alt 同期成功
        Freee-->>Worker: 200 OK
        Worker->>Journal: status=EXPORTED / Outbox=PROCESSED
    else 一時障害・レート制限 (RETRYABLE_TRANSIENT)
        Freee-->>Worker: 5xx / 429
        Worker->>Journal: status=EXPORT_FAILED / attemptCount++ / nextRetryAt 設定
        Worker->>Alert: 滞留アラート (30分継続で IR-002)
    else 3回失敗 or NON_RETRYABLE
        Worker->>Journal: DLQ 隔離 (破棄禁止 / needsTriage=true)
        Worker->>Alert: DLQ 件数 > 0 アラート (15分以内 / IR-003)
    end
    deactivate Worker

```
