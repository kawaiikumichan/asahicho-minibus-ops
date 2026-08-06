# Operations Manual (運用マニュアル)

## Document Control (文書管理情報)

| Item | Value |
|---|---|
| **Document Owner** | SRE Team |
| **Version** | v1.0 |
| **Review Cycle** | Quarterly |
| **Next Review** | 2026-11-06 |
| **Approved By** | CTO |
| **Effective Date** | 2026-08-06 |

### Change History (改訂履歴)

| Version | Date | Summary |
|---|---|---|
| v1.0 | 2026-08-06 | Initial Release |

---

本マニュアルは、ASAHI Coach App の SRE および運用担当者向けに、商用環境における定常業務およびインシデント発生時の対応手順（Runbook）を定義した標準運用手順（Standard Operating Procedure: SOP）です。

**※ 運用上の前提 (Operational Premise)**
本マニュアルは提出された設計書・ADR・実装成果物を基に作成した運用手順書であり、実際の Production 環境の構成・IAM 権限・監視設定は各環境の最新構成を正 (Source of Truth) とします。

**※ 用語定義 (Terminology)**
- **JournalEntry**: 会計仕訳（すべての財務変動を記録する不変の元帳データ）
- **Invoice**: 請求書（保護者へ提示する請求データ）
- **Deal**: 取引（会計システム・freee等へ連携する際の取引データ）

**※ Runbook ID 命名規則**
本マニュアルの手順には一貫性を持たせるため、以下の ID を付与しています。
- **IR-xxx**: Incident Response (障害対応)
- **MO-xxx**: Monthly Operations (月次運用)
- **DR-xxx**: Disaster Recovery (災害・緊急復旧)
- **RM-xxx**: Routine Maintenance (定期保守)

---

## 1. Incident Response (インシデント対応)

インシデント発生時は、以下の Severity に基づきエスカレーションと対応を行います。

| Severity | 定義・例 | 目標復旧時間 (SLA) | エスカレーション先 |
|---|---|---|---|
| **P1** | 決済機能の停止、全社的なデータ破壊等 | 30分以内 | CTO、Security Architect |
| **P2** | freee API 連携の継続的失敗など、特定機能の広範な障害 | 4時間以内 | 開発チームリード |
| **P3** | 一部ユーザーでのデータ不整合、手動消込の滞留など | 次営業日以内 | SRE 担当者 |

### IR-001: Payment Webhook Failure (決済通知の受信失敗)
- **Severity**: P2
- **RACI**: Executor: SRE / Approver: CTO / Notify: Accounting Admin
- **Prerequisites**: Production Project Viewer, Cloud Logging 閲覧権限, Secret Manager Accessor
- **Escalation Criteria (P2 → P1)**: 障害が 30 分以上継続、全体の決済失敗率が 10% 超過、または全ユーザーに影響が及ぶ場合。
- **症状:** Stripe Dashboard または GMO コンソール上で Webhook 配信エラー (5xx/Timeout) が発生している。
- **原因:** 一時的な Cloud Functions のコールドスタート遅延、または freee API 等のダウンタイムによる同期処理のタイムアウト。
- **対応手順:**
  1. Cloud Logging にて `WebhookSecurityError` (署名不一致/改ざん) か `TimeoutError` かを確認する。
  2. 署名エラーが多発する場合は、Secret Key の漏洩を疑い、直ちに CTO へエスカレーションし Secret Manager を確認する。
  3. `TimeoutError` の場合、Idempotency 制御 (ADR-010) が効いているため、決済プロバイダ側から手動で Webhook を再送 (Retry) する。
- **Success Criteria:**
  - `Error Count (Webhook) = 0` に復帰していること。
  - 対象 Invoice のステータスが `PAID` に遷移し、紐づく `JournalEntry` が生成されていること。
- **Post Incident Actions:**
  - Incident Management System (Jira/PagerDuty 等) のチケットを更新・クローズ。
  - （頻発時）RCA の実施および ADR / Risk Register の更新。

### IR-002: Freee API Export Failure (会計エクスポート失敗)
- **Severity**: P2
- **RACI**: Executor: SRE / Approver: Accounting Admin / Notify: Development Team Lead
- **Prerequisites**: Production Project Viewer, Firestore 更新権限
- **症状:** Invoice が `PAID` になったが、freee 会計側に Deal (取引) / JournalEntry が反映されていない。
- **原因:** freee API のメンテンスダウン、または一時的な API レート制限への到達。
- **対応手順:**
  1. Firestore の `JournalEntry` コレクションを確認し、ステータスが `EXPORT_FAILED` または保留状態になっているエントリを特定する。
  2. バッチ再処理スクリプト (`npm run tools:retry-export`) を実行し、未送信の JournalEntry を再送する。
- **Success Criteria:**
  - `Pending JournalEntry = 0` (すべて `EXPORTED` に遷移) となっていること。
  - freee 会計上に正しい内容の Deal が生成されていること。
- **Post Incident Actions:**
  - Incident Management System のチケットを更新・クローズ。

---

## 2. Monthly Operations (月次定常業務)

### MO-001: Month End Close (月次締め処理)
- **RACI**: Executor: Accounting Admin / Notify: SRE, CTO
- **Prerequisites**: Admin Portal 操作権限
1. 管理画面（Admin Dashboard）から「月次締め (Lock Period)」を実行する。
2. システムが前月の全 `JournalEntry` を集計し、RFC8785 Canonical Hash (`periodHash`) を生成・Firestore へ記録する。
3. 期間ステータスが `LOCKED` に遷移したことを確認する。
- **Success Criteria:**
  - 期間ステータスが `LOCKED`。
  - `periodHash` が生成済。
  - ロック対象期間の `JournalEntry` 件数がバッチ処理の集計結果と完全に一致していること。
  - 実行者の Audit Log が生成済。

### MO-002: Manual Reconciliation (手動消込と残高調整)
- **RACI**: Executor: Accounting Operator / Approver: Accounting Admin
- **Prerequisites**: Admin Portal 操作権限
1. 管理画面の「要確認入金一覧」から該当レコードを開く。
2. オペレーターが実際の銀行口座と照合し、紐付けるべき Invoice を手動で選択 (Match) して「消込実行」を押下する。
3. 実行時、システムは以下の情報を確実に監査ログ (Audit Log) へ記録する。
   - `Operator ID`, `Approver ID`, `Timestamp`, `Invoice ID`, `Reason`
- **Success Criteria:**
  - 対象 `Invoice` が `PAID` に遷移し、Audit Log に上記全 5 項目が記録されていること。

---

## 3. Disaster Recovery (緊急復旧)

### DR-001: Emergency Rollback (システムロールバック)
- **Severity**: P1
- **RACI**: Executor: SRE / Approver: CTO / Notify: 全社
- **Prerequisites**: Vercel/Firebase Deployer 権限、Secret Manager Admin

| 障害スコープ | ロールバック対象 | 実行ツール/方法 |
|---|---|---|
| フロントエンドのバグ | **Hosting** (WebUI) | Vercel ダッシュボードから前バージョンへ Revert |
| APIのロジックエラー | **Functions** (Backend) | Firebase CLI `firebase deploy --only functions` |
| アクセス制御のバグ | **Firestore Rules** | Firebase CLI または GCP Console より適用 |
| クエリエラー | **Indexes** | `firebase deploy --only firestore:indexes` |
| キーの漏洩・不具合 | **Secrets** | Google Cloud Secret Manager のバージョン切替 |
| 深刻なデータ不整合 | **Database** | 下記「DR-002」へ移行 |
- **Post Incident Actions:**
  - RCA の作成と CTO への提出。Incident Management System のチケット更新。

### DR-002: Firestore Data Recovery (PITR リストア)
- **Severity**: P1
- **RACI**: Executor: SRE / Approver: CTO, Security Architect / Notify: Accounting Admin
- **Prerequisites**: GCP Project Owner 権限

1. **Restore**: Google Cloud Console より、障害直前のタイムスタンプを指定し、**一時的な復旧プロジェクトへリストア**する。
2. **Read Only Validation**: リストアされた環境に対して、読み取り専用権限で検証する。
3. **Diff**: SRE チームがスクリプトを用いて本番環境とのデータ差分 (Diff) を抽出する。
4. **Approval**: 抽出された Diff 結果を CTO 等へ提出し、マージの承認 (Approval) を得る。
5. **Verification**: 差分レコード件数、Hash 値、サンプリングによる照合を行い、意図した差分のみであることを最終確認する。
6. **Merge**: 承認・検証された差分のみを本番へマージする。
- **Rollback Plan (マージ失敗時):**
  - マージが失敗した、または Verification で異常が検知された場合は、**復旧プロジェクトを直ちに破棄し、本番環境への変更は一切行わない**。
- **Post Incident Actions:**
  - BCP/DR 計画の見直しおよび RCA の実施。

---

## 4. Routine Maintenance (定期メンテナンス)

日常運用において、SRE は以下の監視対象 (Monitoring Scope) と定期タスクを実行します。

### Monitoring Scope (日常の監視対象)
| Service | Monitoring Metrics |
|---|---|
| Cloud Functions | Error Rate / Latency |
| Firestore | Read・Write Usage / Query Latency |
| Stripe Webhook | Delivery Failure / Timeout Count |
| freee API | Export Failure / Rate Limit Reached |
| Secret Manager | Secret Version Rotation Status |

### RM-001: 定期メンテナンスタスク一覧
| 頻度 | タスク内容 | Success Criteria |
|---|---|---|
| **Daily** | Cloud Error Reporting の確認 | `Error Count (未処理 5xx) = 0` |
| **Weekly** | Secret 期限・ローテーション確認 | 30日以内に期限切れとなるシークレット数 = 0 |
| **Monthly** | Restore Drill (リストア訓練) | 復元から Diff 抽出までの手順完遂時間が 15分以内 |
| **Quarterly** | RBAC 監査 および Runbook レビュー | 不要権限の削除完了、および本 Runbook のアップデート完了 |
| **Yearly** | BCP見直し・DR演習・権限全件レビュー | DR 演習の完遂、および全アクセス権限の正当性確認完了 |

---

## 5. KPI / SLO (Operational Targets)

運用においてシステムが維持すべきサービスレベル目標（SLO）と、その測定指標（SLI）です。

| SLO | Target | SLI (測定指標) |
|---|---|---|
| Payment Success | ≥ 99.9% | 成功決済数 ÷ 全決済リクエスト数 |
| Webhook Success | ≥ 99.9% | 成功Webhook処理数 ÷ 全Webhook受信数 |
| MTTR (Severity P1) | < 30 min | P1障害の復旧完了時刻 − 障害検知時刻 |
| Restore Drill | < 15 min | リストア開始からDiff抽出完了までの所要時間 |
| Error Budget | 0.1% / 月 | 月間の全リクエストに対する 5xx エラー率 |
