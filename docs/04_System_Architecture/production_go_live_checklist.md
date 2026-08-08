# Production Go-Live Checklist

本チェックリストは、Production Go-Live の判定に必要な確認項目を定義するものである。Evidence Verification は提出された成果物に基づくレビュー結果を示し、Production 環境への実際の適用状況を保証するものではない。Production 環境での最終確認は Go-Live Approval 時に実施する。

すべての Blocker 項目が `☑ Verified` (PASS / 証跡確認済) となるまで、商用トラフィックの受け入れは禁止される。本リストの判定基準は `operational_risk_register.md` と連動している。

### Status Legend
- `☐ Not Verified`: 証跡未確認、またはテスト/デプロイ未実施
- `☑ Verified`: 証跡確認済、または本番適用確認済
- `⚠ Accepted Risk`: リスク受容済（Approverの承認済）

## 1. Security & Access Control

| Status | Item | Environment | Pass Criteria | Evidence Verification |
|---|---|---|---|---|
| `☐` | **Firestore Rules Deploy** | Production | `firestore.rules` がデプロイされ、ADR-016/017 に準拠していること。 | 提出物からルール実装を確認済。Productionデプロイは未確認 |
| `☐` | **Custom Claims Provisioning** | Production | 初期管理者の Custom Claims (`organizationId`, `role: ADMIN`) が設定可能なこと。 | 提出物から実装仕様を確認済。Production設定は未確認 |
| `☑` | **Secret Manager Configuration** | Production | API Keys / Webhook Secrets が Secret Manager で管理され、無停止ローテーションが可能なこと。 | 🚨 **Blocker (RSK-003)**: 提出された Evidence Report により Verified (Production証跡確認済) |

## 2. Infrastructure & Data Integrity

| Status | Item | Environment | Pass Criteria | Evidence Verification |
|---|---|---|---|---|
| `☐` | **Composite Index Deploy** | Production | `firestore.indexes.json` がデプロイされ、必要なインデックスが構築されていること。 | 提出物から定義を確認済。Productionデプロイは未確認 |
| `☐` | **Backup & PITR Enabled** | Production | Firestore の PITR および日次バックアップが有効化されていること。 | ⚠ **Evidence Not Verified**: 宣言のみ確認済 |
| `☑` | **Restore Drill Completed** | UAT | 15分以内にデータを復元し、整合性が確認できること。 | 🚨 **Blocker (RSK-004)**: 提出された Evidence Report により Verified (RFC8785 Diff実証済) |

## 3. Integrations

| Status | Item | Environment | Pass Criteria | Evidence Verification |
|---|---|---|---|---|
| `☐` | **Stripe Live Keys Configured** | Production | Stripe の本番キーが設定され、Webhook エンドポイントが疎通すること。 | 提出物からSandbox構成の証跡を確認済。Production構成は未確認 |
| `☐` | **GMO Aozora Live Configured** | Production | GMO の本番証明書とエンドポイントが構成されていること。 | 提出物からSandbox構成の証跡を確認済。Production構成は未確認 |
| `☐` | **freee OAuth Live App Configured**| Production | freee 会計アプリの本番用 OAuth 認証が許可されていること。 | 提出物からSandbox構成の証跡を確認済。Production構成は未確認 |

## 4. Observability & Monitoring

| Status | Item | Environment | Pass Criteria | Evidence Verification |
|---|---|---|---|---|
| `☐` | **Structured Logging Active** | Production | JSON 構造化ログが Cloud Logging 上で確認できること。 | 🚨 **Operational Readiness Blocker**: 提出された成果物から設定証跡を確認できず |
| `☑` | **Correlation ID Injection** | Production | Webhook 受信から JournalEntry まで同一 ID でトランザクションを追跡できること。 | 🚨 **Blocker (RSK-001)**: 提出された Evidence Report により Verified (構造化ログ・SpanID実装済) |
| `☐` | **Alert Policies Configured** | Production | 5xx エラー急増等の Cloud Error Reporting アラートが設定されていること。ADR-022 2.5 の必須アラート（署名検証失敗 / DLQ 件数 / Outbox 滞留 / `EXPORT_FAILED` 滞留 / 障害由来 DENY 率）をすべて含むこと。 | 🚨 **Operational Readiness Blocker**: 提出された成果物から設定証跡を確認できず |

## 5. Reliability & E2E Validation

| Status | Item | Environment | Pass Criteria | Evidence Verification |
|---|---|---|---|---|
| `☐` | **Cloud Functions Retry Enabled** | Production | 一時エラーに対する Backoff Retry が有効化されていること。 | ⚠ **Evidence Not Verified**: 提出物から設定証跡を確認できず |
| `☐` | **Full E2E Emulator Pass** | Emulator | Attendance〜Accounting 全シナリオの競合・切断テストがPASSし、重大欠陥が0件であること。 | ⚠ **Evidence Not Verified (RSK-002)**: 提出された成果物から実証記録を確認できず |
| `☐` | **Ack Boundary Verified (ADR-022)** | Emulator | 耐久化コミット失敗時に Webhook が 5xx を返し、プロバイダ再送で欠落 0 件に回復すること。署名検証失敗が `PaymentEvidence` に `FAILED` として残ること。 | ⚠ **Evidence Not Verified (RSK-010)**: 仕様は ADR-022 で規定済。実証記録は未提示 |
| `☐` | **DLQ Replay Drill** | UAT | 3 回失敗イベントが破棄されず DLQ に隔離され、IR-003 の Manual Replay で二重計上なく復旧できること。 | ⚠ **Evidence Not Verified (RSK-008)**: IR-003 手順は策定済。実施記録は未提示 |

---

## 6. Go-Live Approval

上記の Blocker 項目がすべて解消され、関連する証跡が Production 環境等において最終確認された場合のみ Go-Live を承認する。

| Role | Name | Result | Date |
|---|---|---|---|
| **CTO** | — | PASS / HOLD | — |
| **SRE** | — | PASS / HOLD | — |
| **Security Architect** | — | PASS / HOLD | — |
