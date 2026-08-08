# ASAHI CLUB Architecture Governance & Design Principles (120-Point Enterprise & Operational Edition)

Status: Approved
Version: 2026.07
Owner: Engineering & Operations Team

This document is the supreme architecture governance specification and common design constitution for all ASAHI CLUB domain specifications.

---

本ドキュメントは、ASAHI Coach App の各ドメインライフサイクル（Family, Invoice, Wallet, Attendance, Notification, Tournament, Doctor）を貫く **「アーキテクチャ統制および設計憲章 (Architecture Governance & Design Principles - 120-Point Enterprise & Operational Edition)」** です。
エンジニアリングチームおよび業務委託・外部ベンダーは、新しい機能・データモデル・サービスを開発または修正する際、必ず本憲章を最上位原則として順守しなければなりません。

---

## 1. 25大アーキテクチャ共通設計原則 (The 25 Design Commandments)

### 1.1 Documentation First (仕様書駆動開発原則)
- 実装コードを修正・追加する前に、必ず本ディレクトリ (`docs/architecture/`) 内の公式仕様書 (Single Source of Truth) を策定・更新しなければなりません。
- 「コードが仕様である」という場当たり的な開発を排し、常に仕様書が先行する **Documentation First** を徹底します。

### 1.2 SSOT (Single Source of Truth / 真実情報源の絶対性)
- 各ドメインには唯一の正データ（SSOT）が存在します。
  - **Family**: `users.familyId`
  - **Wallet**: `app-activity-fee-transactions` (`ActivityFeeTransaction`)
  - **Invoice**: `invoices`
  - **Attendance**: `app-attendance-records` (`attendance_records`)
- パフォーマンス向上のためのキャッシュや派生データ（`families.memberIds` や画面表示用の残高等）は、いかなる瞬間も SSOT から 100% 決定論的に再計算・再構成できる必要があります。

### 1.3 Event Sourcing / Immutable Ledger (不変元帳・追跡可能設計)
- 金銭や権利変更に関わる重要トランザクション（活動費ウォレット、会費請求相殺、退会、代表権移譲）は、既存データの変更・上書きによって履歴を消去することを禁じます。
- 取引および履歴は追加専用の **不変元帳 (Immutable Ledger / Event Sourcing)** として保存し、訂正や取消は必ず反対符号の **逆仕訳 (Reversal)** を発行して監査チェーン (`relatedTransactionId`) を維持します。

### 1.4 Layered Architecture (レイヤード責務分離と CQRS / UseCase 分離)
- UI層（Svelte コンポーネント）・Application Service (UseCase) 層・Domain Service 層・Repository 層を明確に分離し、レイヤー境界を超えた直接I/Oやビジネスロジックの漏出を禁じます。

### 1.5 Dependency Rule (公開 Service インターフェース依存と循環参照防止)
- ドメイン間の依存は **公開 Service インターフェース (`*Service.ts` / UseCase)** に限定します。
- **Repository 間の直接依存 (`InvoiceRepository -> WalletRepository` 等) は厳格に禁止** します。

### 1.6 Domain Isolation (ドメイン間のコレクション直接書き込み禁止)
- あるドメインが他ドメインの Firestore コレクションや DB を直接更新・削除することを厳格に禁止します。
- 他ドメインのデータを変更・操作する場合は、**必ず相手ドメインの Service 公開メソッドを経由**しなければなりません。

### 1.7 Transaction Boundary & Event Firing Order (アトミック更新とイベント発火順序)
- 複数のドメインおよびコレクション間における一貫性が求められる操作は、必ず **Firestore Transaction (`runTransaction`)** を用いてアトミックに実行することを義務付けます。
- **イベント発火順序規約 (Event Firing Order)**:
  ```text
  [1. Atomic Firestore Transaction] -> [2. Commit] -> [3. Emit Domain Event] -> [4. AuditLog Record] -> [5. Notification Delivery] -> [6. Async Doctor Queue / Job]
  ```

### 1.8 Repository MUST NOT & Interface Abstraction (抽象リポジトリ規定)
- `Repository` は `Persistence Layer, Object Mapping, Query, Transaction Execution` に責任を持ち、残高計算・業務ルール判定・通知送信・採番・Doctor診断を含めることを厳禁します。
- **Repository Interface 抽象化規定**: 各ドメインは必ず `interface AttendanceRepository`, `interface InvoiceRepository`, `interface WalletRepository` を定義し、具象実装ではなくインターフェースに依存しなければなりません。

### 1.9 Service MUST NOT (サービス層の禁止事項)
- `Service` はビジネスロジック・採番・ドメインバリデーションに責任を持ち、**Firestore SDK (`collection()`, `doc()`, `runTransaction()` 等) の直接インポートおよび実行を厳格に禁止**します（ブートストラップ・DI 例外を除く）。

### 1.10 Firestore Collection Naming & Versioning Policies (4層バージョニング統制)
- 命名規約: 小文字のスネークケースまたはアプリ接頭辞（例: `app-activity-fee-transactions`, `invoices`, `app-attendance-records`）。
- **4層バージョニングポリシー**:
  - `Specification Version`: 仕様書バージョン (`v2026.07`)
  - `Schema Version`: データモデル不変スキーマ (`schemaVersion: 1`)
  - `API Version`: REST / IPC インターフェース (`v3`)
  - `Migration Version`: マイグレーション日付タグ (`20260730`)

### 1.11 Audit Log Policy (不変監査ログ要件)
- クリティカルな状態変更は、全操作において `audit_logs` コレクションへの書き込みを必須とします（`eventType`, `familyId`, `actorId`, `details`, `before`/`after`）。

### 1.12 Doctor Release Gate & Plugin Architecture (プラグイン化品質保証)
- **Doctor Plugin Architecture**: 条件分岐地獄を排し、独立したプラグイン規則 (`interface DoctorRule`) を実装・登録する拡張モデルとします。
- 本番デプロイ時、以下の合格条件を満たさなければ CI/CD を即時シャットダウンします:
  ```text
  CRITICAL = 0, ERROR = 0, WARNING = 0, Audit Coverage = 100%, UAT = PASS, E2E = PASS
  ```

### 1.13 Idempotency & Transaction Keys (べき等性とトランザクションキー)
- バッチ処理や自動相殺における二重実行・二重請求を完全に防ぐため、業務キーである `transactionKey` / `invoiceKey` を事前に確定・検証してから実行します。
  - **Invoice**: `INV-YYYYMM-familyId`
  - **Wallet**: `OFF-INV-YYYYMM-XXX`, `RIDE-YYYYMMDD-XXXX`, `DEP-YYYYMM-XXXX`

### 1.14 Domain Event & Decoupled Messaging (イベント駆動アーキテクチャ)
- サービス間の呼び出し地獄を回避するため、ドメイン操作完了時に標準化された **Domain Event** (`RideCompleted`, `InvoicePaid`, `FamilyMerged`) を発行 (`emit`) し、監視・通知サービスはそれを購読 (`subscribe`) します。

### 1.15 Event Sourcing Metadata (イベント追跡メタデータ)
- 全ドメインイベントは因果関係を追跡可能にする 3 大 ID を保持しなければなりません:
  - `eventId`: イベント単体の UUID
  - `correlationId`: セッション全体の UUID
  - `causationId`: トリガーとなった親イベントやコマンドの UUID

### 1.16 Read Model / CQRS Projection (読み取り専用モデルの分離)
- SSOT (Write Model) の整合性を維持しつつ画面表示 (`Dashboard`, `Monthly Report`) を高速化するため、Domain Event を受け取って構築する **Read Model / Projection** の使用を許可します。

### 1.17 Standard Domain Package Structure (標準パッケージ構造規約)
- 各ドメインは以下の統一されたディレクトリ構成に従わなければなりません:
  `model/`, `repository/`, `service/`, `usecase/`, `dto/`, `events/`, `rules/`, `doctor/`, `uat/`, `tests/`

### 1.18 Release Manifest (リリース監査マニフェスト)
- リリース時は必ず CI/CD において **`release-manifest.yaml`** を生成・検証し、「どのアーキテクチャ・スキーマ・Doctor・ADR バージョンをデプロイしたか」を厳格に追跡します。

### 1.19 Aggregate Boundary (DDD集約境界の明文化)
- トランザクション境界を明確化するため、システム全体の集約境界（Aggregate Root）を定義し、1つのアトミックトランザクションにおける更新対象を集約内部に限定します。
  - **Family Aggregate**: `Family` (`app-families`) を Root とし、`users`, `withdrawalRequest` を含む。
  - **Invoice Aggregate**: `Invoice` (`invoices`) を Root とし、`invoiceItems` を含む。
  - **Wallet Aggregate**: `ActivityFeeTransaction` (`app-activity-fee-transactions`) を独立した Ledger Root とし、派生残高を管理する。
  - **Attendance Aggregate**: `Schedule` (`schedules`) を Root とし、`attendance_records`, `rides` を含む。

### 1.20 Domain Invariants (常時成立条件の公式定義)
- ドメインイベントや状態変化にかかわらず、**「システム内で常時100%成立していなければならない絶対条件 (Domain Invariants)」** を規定します。Doctor はこれら Invariants を常時監査します。
  - **`INV-FAM-001`**: 代表保護者 (`billingUserId`) は必ず家族のメンバー ID リスト (`memberIds`) の中に存在する (`billingUserId ∈ memberIds`)。
  - **`INV-WAL-001`**: 家族の有効ウォレット残高は、全確定 Ledger トランザクションの合計値と完全一致する (`confirmedBalance == SUM(confirmed ledger)`)。
  - **`INV-INV-001`**: 請求書合計額は、明細項目合計額と一致する (`invoiceTotal == SUM(invoiceItems.amount)`)。

### 1.21 Concurrency Policy (楽観的排他制御・競合解決)
- 代表権移譲・退会承認・請求書相殺などの競合する同時更新処理においては、必ず **楽観的排他制御 (Optimistic Concurrency Control: OCC)** を実装します。
- 各ドキュメントに `schemaVersion`, `updatedAt`, およびバージョン番号（または `etag` / `lastUpdateVersion`）を持たせ、競合検出時は更新をロールバックした上でエラーを返します。

### 1.22 Retry, DLQ & Compensation Policy (イベント失敗時の回復と補償戦略)
- コミット後の非同期通知や外部サービス連動が失敗した場合の回復戦略を規定します（失敗分類・可視化義務の詳細は **ADR-022** を参照）:
  - **Failure Classification (失敗分類の先行義務)**: 再試行するか否かは必ず ADR-022 2.1 の `failureClass` 判定に基づいて決定します。分類不能な失敗を「一時障害」と見なして無限に再試行することを禁じます。
  - **Retry Policy**: `RETRYABLE_TRANSIENT` / `CONFLICT` に限り、指数バックオフ (Exponential Backoff) で最大 3 回再試行。
  - **Dead Letter Queue (DLQ)**: 3 回失敗したメッセージ、および再試行不可 (`NON_RETRYABLE_*` / `UNKNOWN`) のメッセージは DLQ に隔離します。**DLQ からのメッセージ破棄を禁止**し、`attemptCount` / `lastError.failureClass` / `lastError.code` / `lastAttemptAt` を必ず永続化します。DLQ 件数が 1 件以上となった時点で管理アラートを発報し、復旧手順は Operations Manual **IR-003** に従います。
  - **Compensation Transaction (補償トランザクション / Saga)**: 配車クレジット付与後に請求バッチが致命的エラーで失敗した場合、自動的に逆仕訳 (`reversal`) を発行して状態を回復する。補償の発行自体が失敗した場合は、二重補償を避けるため自動再試行せず DLQ 隔離＋会計管理者エスカレーションとします。
  - **Poison Event Handling**: 不正形式のイベントメッセージは再試行対象外 (`NON_RETRYABLE_VALIDATION`) とし、**破棄・スキップではなく DLQ へ隔離**した上で管理アラートを発報します（サイレントロストの禁止）。
  - **No Silent Success (成功マスクの禁止)**: 失敗したイベントを `PROCESSED` へ遷移させること、および失敗を既定値・空応答で覆い隠すことを禁じます。

### 1.23 Domain Event Versioning (イベントスキーマの互換性)
- すべての Domain Event には **`eventVersion`** (例: `eventVersion: 2`) を必須定義します。
- スキーマ変更時はバージョンを上げ（例: `RideCompletedV1` -> `RideCompletedV2`）、サブスクライバー側で両バージョンを安全にハンドリング可能にします。

### 1.24 Security & RBAC Architecture (権限分離アーキテクチャ)
- Firebase Authentication / Firestore Security Rules に基づき、ロールごとの権限境界を定義します:
  - **`Guardian / Parent`**: 自家族アカウント情報および自子弟の出欠表示・締切前編集のみ許可。
  - **`Coach`**: 担当チームスケジュール管理、出欠強制更新 (`override`)、配車割り当て許可。
  - **`Admin`**: クラブ全体管理、請求書発行・消込、代表権強制移譲、退会処理許可。
  - **`CI / Doctor / Batch`**: システムサービスアカウント。非破壊の読取・整合性診断・自動修復および CI リリースゲート審査を許可。

### 1.25 Configuration & Observability Architecture (構成管理と運用監視)
- **Configuration Architecture**: ビジネスルールパラメータ（例: 配車補助単価 200円/km 等）をコードにハードコードせず、以下の4層で設定管理します:
  `System Settings`, `Organization Settings` (`app-organizations` コレクション), `Feature Flags`, `Environment Variables`
- **Observability (監視・メトリクス・トレーシング)**: 監査ログ (`audit_logs`) とは別に運営稼働監視を行います:
  - **Metrics**: `Doctor Execution Time`, `Notification Queue Length`, `Invoice Batch Generation Time`, `Wallet Offset Execution Duration`
  - **Tracing / Logging / Alerts**: SLO 超過やエラー発生を Cloud Monitoring 等へアラート発報。

---

## 2. 全体アーキテクチャ・依存関係およびイベントフロー (Domain Dependency Flow)

```text
========================================================================================
                 ASAHI CLUB DOMAIN DEPENDENCY & EVENT FLOW (120-Point Edition)
========================================================================================

                  [ 静的マスタ参照 (Static Master Reference) ]
                                 │
                                 ▼
                             Family (01)
                     [家族・保護者・選手アカウント SSOT]
                                 │
                   +-------------+-------------+
                   │                           │
  [同期サービス呼出]│                           │ [静的マスタ参照]
  (Sync Call)      ▼                           ▼
            Attendance (04)               Invoice (02)
     [出欠・練習・試合イベント SSOT]     [月次会費・遠征費請求・自動相殺 SSOT]
                   │                           │
                   ▼ (Sync Call)               ▲
                Ride (04)                      │
     [遠征配車・車出し・同乗者リスト SSOT]          │ [同期サービス呼出:
                   │                           │  WalletService.offsetInvoice]
                   ▼ (Emit Domain Event:       │
                      RideCompleted)           │
                Wallet (03) ───────────────────+
       [活動費・配車補助・預り金・残高元帳 SSOT]
                   │
                   +---------------------------+
                   │                           │
                   ▼ (Emit Event:              ▼ (Emit Event:
                      WalletCreditAdded)          InvoiceCreated/Paid)
                   +-------------+-------------+
                                 │
                                 ▼ [非同期イベント購読 (Async Subscribe)]
                         Notification (05)
                [LINE / Mail / Push 通知配信・既読 SSOT]

 ──────────────────────────────────────────────────────────────────────────────────────
                                 ▲
                                 │ 監視・整合性監査・リリースゲート
                          Doctor Diagnostics (07)
               [全ドメイン横断品質保証 (Supreme QA Engine)]
========================================================================================
```

---

## 3. Architecture Lifecycle Governance (新規開発 6 ステップルール)

新たな機能やドメインを追加する場合は、以下の 6 ステップを順序どおり完了しなければ実コードの修正に着手してはなりません。

```text
  ① ADR (Architecture Decision Record) を策定し設計判断 (Why) を記録する
  ② Architecture Specification (公式ライフサイクル仕様書) を作成・承認する
  ③ Domain Rules & SSOT を明文化する
  ④ Doctor 診断項目 (DOC-xxx) を登録・定義する
  ⑤ E2E 結合 UAT シナリオ (SCENARIO-xxx) を策定・自動化する
  ⑥ 公式 Release Gate 審査パイプラインへ統合する
```

---
*ASAHI Coach App Architecture Governance & Design Principles - Approved by Engineering & Operations Team*
