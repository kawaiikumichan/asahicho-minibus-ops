# Implementation Plan: Remediation of P0 Operational Blockers

本計画は、ASAHI Coach App の本番稼働前審査（Production Readiness Review）にて「Blocker（必須解消事項）」として特定された P0 リスク（RSK-001, RSK-003, RSK-004）を完全に解消するための具体的な実装方針と手順を定義するものです。

## User Review Required

> [!IMPORTANT]
> - 本計画は基盤（オブザーバビリティ、セキュリティ、SRE運用）に関わる実装を含みます。凍結済みのドメインモデル（ADR-001〜ADR-021）には一切変更を加えません。
> - 現在の `club-app` リポジトリに Firebase Functions のディレクトリが存在しないため、今回は Next.js アプリケーション内の共通ライブラリ (`src/lib`) および管理用スクリプト (`scripts/`) として実装を行います。

## Proposed Changes

---

### 1. RSK-001: Correlation ID (オブザーバビリティの強化)

Webhook 受信から `JournalEntry` 生成までの全トランザクションを追跡可能にするため、Node.js の `AsyncLocalStorage` を用いたロギング基盤を実装します。

#### [NEW] `club-app/src/lib/logger/AsyncContext.ts`
- `AsyncLocalStorage` を用いて、現在のリクエストスコープに紐づく `correlationId` や `tenantId` (`organizationId`) などのメタデータを保持するコンテキストプロバイダを実装します。引回し（Prop-drilling）なしでどの階層からでもアクセスできるようにします。
- **Request Entry Point の生成責務**: Webhook 等のエントリーポイントにおいて、ヘッダー (`X-Correlation-ID`) が存在すればそれを再利用し、存在しなければ `UUID v4` を新規生成するロジックを実装します。

#### [NEW] `club-app/src/lib/logger/CloudLogger.ts`
- 構造化 JSON ロガー（Google Cloud Logging 形式準拠: `logging.googleapis.com/trace` など）を実装します。
- ログ出力時に `AsyncContext` から情報を自動的に抽出し、ペイロードにインジェクトする仕組みを構築します。

---

### 2. RSK-003: Secret Manager Integration & Rotation (ゼロダウンタイム・シークレットローテーション)

API Keys (Stripe Webhook Secret 等) をハードコードや `.env` のみに頼らず、Google Cloud Secret Manager から安全に取得し、プロセス再起動なしでローテーション（再読込）できる機構を実装します。

#### [NEW] `club-app/src/lib/secrets/SecretManagerService.ts`
- Google Cloud Secret Manager API（ローカル環境用シミュレーションフォールバック付き）を呼び出し、最新のシークレットバージョンを取得するサービス。
- キャッシュ機構（TTL付き）を設け、パフォーマンス低下を防ぎつつ一定時間（例: 5分）ごとにバックグラウンドで再読込を行う「ゼロダウンタイム・ローテーション」を実現します。

> [!WARNING]
> シミュレーション実装だけでは Risk Register 上の Blocker は解消されません。最終的な Go-Live Approval に向けては、Staging / Production 環境の Secret Manager での**実証確認 (Production Verification)** が必須となります。
> - **Production Verification Requirements**:
>   - Secret Manager への本番キー登録確認
>   - IAM Access Control 設定確認
>   - アプリケーションからの取得成功確認
>   - 稼働中の Secret Version 切替および無停止動作確認

---

### 3. RSK-004: Firestore Restore Drill Script (データ復旧用 Diff エクストラクター)

オペレーションマニュアル (DR-002) で定義された厳格なリストア手順「Read Only Validation -> Diff -> Approval -> Verification -> Merge」の安全ゲートを通過するための、SRE 用差分抽出スクリプトを実装します。

#### [NEW] `club-app/scripts/dr/restore-diff-extractor.ts`
- 本番プロジェクトと復旧用一時プロジェクト（リストア先）のデータを比較検証するスクリプト。
- ASAHI Coach App のライフサイクル全域（`Attendance` → `Ride` → `WalletLedgerEntry` → `Invoice` → `PaymentRecord` → `JournalEntry` → `AuditLog`）の不変レコードを対象とし、レコード件数・ハッシュ値の突合（Diff）を行います。
- **Hash Verification**: 突合には `Canonical JSON (RFC8785) -> SHA-256 -> Domain Record Hash` の不変性検証プロセスを用います。
- Merge 前の `Verification` ステップで経営陣等に提出できる検証レポートを標準出力します。

---

## Verification Plan

### Evidence Matrix

Go-Live Checklist の判定基準を満たすため、以下の Matrix に沿って証跡 (Evidence) を生成し `walkthrough.md` に記録します。

| Risk ID | Verification Test | Required Evidence | Owner |
|---|---|---|---|
| **RSK-001** | Mock request simulation | 構造化 JSON ログのサンプル (Trace ID付) | SRE |
| **RSK-003** | Secret rotation test (Local & Prod) | Secret Manager Version List, IAM Policy Export, Application Access Log, Rotation Execution Log | Security Architect |
| **RSK-004** | Restore diff test (UAT/Mock) | 5大ドメインの Diff レポート出力結果 (RFC8785 Hash 比較) | SRE |

### Manual Verification
実装と動作確認が完了次第、それぞれの結果を実証証跡（Evidence Report）として `walkthrough.md` にまとめ、Reviewer へ提示します。

---

## Implementation Completion Criteria

Go-Live 承認にあたり、以下の全条件（Criteria）を満たしていること。これらが充足されて初めて、Go-Live Checklist の Status を `[☑ Verified]` に更新できます。

### RSK-001 (Correlation ID)
- [ ] `CloudLogger` による構造化 JSON ログが生成されること。
- [ ] Webhook 等のエントリーポイントで生成または継承された Correlation ID が、最終的な `JournalEntry` 生成処理まで一貫して伝播すること。
- [ ] Cloud Logging 上で Correlation ID による検索が可能であること (Traceability の保証)。

### RSK-003 (Secret Rotation)
- [ ] `SecretManagerService` が実装され、キャッシュの TTL 自動更新機構が動作すること。
- [ ] (Production) Secret Manager の本番構成および IAM ポリシーエクスポートが提出されていること。
- [ ] (Production) プロセス無停止でのローテーション実行ログ（Rotation Execution Log / Access Log）が提出されていること。

### RSK-004 (Restore Drill)
- [ ] リストア Diff 抽出スクリプト (`restore-diff-extractor.ts`) が実装されていること。
- [ ] `WalletLedgerEntry`, `Invoice`, `JournalEntry` 等の複数ドメインにわたる Diff レポートが生成されること。
- [ ] DR-002 の Approval ワークフローが検証可能であること。
