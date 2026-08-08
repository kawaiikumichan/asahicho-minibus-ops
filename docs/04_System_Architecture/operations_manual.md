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

**※ エラー取り扱いの大前提 (Error Handling Premise)**
本マニュアルの全手順は **ADR-022（エラー伝播・失敗可視化原則）** を前提とします。すなわち、失敗は必ず `failureClass` として分類・記録され、未処理のまま成功応答 (2xx) を返すことはなく、再試行上限到達後のメッセージは破棄されず DLQ へ隔離されていることを前提に調査します。したがって「エラーが見当たらないが数字が合わない」場合は、実装側の ADR-022 違反（握りつぶし）を疑い、RCA 対象としてエスカレーションしてください。

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
- **症状:** Stripe Dashboard または GMO コンソール上で Webhook 配信エラー (4xx/5xx/Timeout) が発生している。
- **原因:** 署名検証失敗（400）、一時的な Cloud Functions のコールドスタート遅延、または受信イベントの耐久化コミット失敗（5xx）。
- **対応手順:**
  1. Cloud Logging にて `failureClass` を確認し、`SECURITY`（署名不一致/改ざん）か `RETRYABLE_TRANSIENT`（`TimeoutError` 等）かを分類する。
  2. `SECURITY` が多発する場合は、Secret Key の漏洩を疑い、直ちに CTO へエスカレーションし Secret Manager を確認する。検証失敗は `PaymentEvidence` (`signatureVerificationResult = FAILED`) に記録されているため、該当レコードを件数・発生源とともに抽出する（ADR-020 / ADR-021）。
  3. `RETRYABLE_TRANSIENT` の場合、未処理の Webhook に対しては 5xx を返す設計（ADR-022 2.3）のためプロバイダ自動再送で回復する。自動再送期限を超えている場合のみ、Idempotency 制御 (ADR-010) を前提に決済プロバイダ側から手動再送 (Retry) する。
  4. **受信欠落の確認 (必須)**: 上記後、決済プロバイダのイベント一覧と `PaymentEvidence` の `providerEventId` を照合し、**履歴に存在しない（ログにも残っていない）欠落イベント**の有無を ADR-013 対査レポートで最終確認する。
- **Success Criteria:**
  - `Error Count (Webhook) = 0` に復帰していること。
  - 対象 Invoice のステータスが `PAID` に遷移し、紐づく `JournalEntry` が生成されていること。
  - 決済プロバイダのイベント件数と `PaymentEvidence` 件数が一致し、欠落 0 件であること。
- **Post Incident Actions:**
  - Incident Management System (Jira/PagerDuty 等) のチケットを更新・クローズ。
  - （頻発時）RCA の実施および ADR / Risk Register の更新。

### IR-002: Freee API Export Failure (会計エクスポート失敗)
- **Severity**: P2
- **RACI**: Executor: SRE / Approver: Accounting Admin / Notify: Development Team Lead
- **Prerequisites**: Production Project Viewer, Firestore 更新権限
- **症状:** Invoice が `PAID` になったが、freee 会計側に Deal (取引) / JournalEntry が反映されていない。
- **原因:** freee API のメンテンスダウン、または一時的な API レート制限への到達。
- **Detection (検知)**: `JournalEntry.status = EXPORT_FAILED` の滞留が 1 件以上かつ 30 分継続した時点で Cloud Monitoring アラートが発報される（人の目視を検知手段としない / ADR-022 2.5）。
- **対応手順:**
  1. Firestore の `JournalEntry` コレクションを確認し、ステータスが `EXPORT_FAILED` または保留状態になっているエントリを特定し、`lastError.failureClass` / `attemptCount` を確認する。
  2. `RETRYABLE_TRANSIENT` の場合のみ、バッチ再処理スクリプト (`npm run tools:retry-export`) を実行し、未送信の JournalEntry を再送する。
  3. `NON_RETRYABLE_VALIDATION` / `NON_RETRYABLE_BUSINESS`（勘定科目マッビング不整合等）の場合は再送しても必ず失敗するため、IR-003 へ移行し `FreeeJournalMapper` の修正を開発チームへエスカレーションする。
  4. 再送スクリプトは、失敗したエントリを `EXPORTED` としてもみ消すことなく、失敗件数を戻り値として報告することを確認する（握りつぶしの遘遭防止）。
- **Success Criteria:**
  - `Pending JournalEntry = 0` (すべて `EXPORTED` に遷移) となっていること。
  - freee 会計上に正しい内容の Deal が生成されていること。
  - 未解決のエントリは DLQ または `needsTriage = true` として可視状態にあり、黙示的に消えていないこと。
- **Post Incident Actions:**
  - Incident Management System のチケットを更新・クローズ。

### IR-003: Outbox / DLQ Stall & Compensation Failure (非同期イベント滞留・補償失敗)
- **Severity**: P2（会計・決済ドメインのイベントを含む場合は P1）
- **RACI**: Executor: SRE / Approver: CTO / Notify: Accounting Admin, Development Team Lead
- **Prerequisites**: Production Project Viewer, Firestore 更新権限, Cloud Logging 閲覧権限
- **対象リスク**: RSK-008（結果整合性障害からの復旧）
- **Detection (検知)**: DLQ 件数 > 0、または Outbox の未処理イベント最古齢 (Oldest Unprocessed Age) > 15 分でアラート。
- **症状:** Invoice は `PAID` なのに Wallet 相殺や仕訳が反映されない、または Ride 完了後に Wallet クレジットが付与されない。
- **原因:** ワーカー停止・再試行上限到達による DLQ 隔離、Poison Event、または補償仕訳 (`reversal`) 発行の失敗。
- **対応手順:**
  1. DLQ 内イベントを `failureClass` で集計し、`RETRYABLE_TRANSIENT`（外部障害回復待ち）と `NON_RETRYABLE_*`（要修正）に仕分ける。
  2. `RETRYABLE_TRANSIENT`: 外部サービスの復旧を確認した上で Manual Replay を実行する。イベントは `eventId` / `idempotencyKey` を保持しているため、再投入による二重計上は発生しない（ADR-006）。
  3. `NON_RETRYABLE_*`: Replay しても必ず再失敗する。原因（スキーマ不整合・不変条件違反・締め済期間への書込み）を特定し、開発チームへエスカレーションする。締め済期間に属する場合は ADR-011 に従い補正仕訳で対応する。
  4. **DLQ イベントの削除は禁止**。業務上不要と判断した場合も、理由・判断者を記録した上で `DISCARDED_BY_APPROVAL` として残す（Approver: CTO）。
  5. 補償仕訳の失敗時は自動再試行せず、Wallet 残高と確定 Ledger 合計の不変条件 (`INV-WAL-001`) を検査してから手動で逆仕訳を発行する（二重補償防止）。
- **Success Criteria:**
  - DLQ 件数 = 0、または残存件のすべてに `needsTriage` チケットが紐づいていること。
  - Outbox の未処理イベント最古齢 < 5 分に復帰していること。
  - `INV-WAL-001` / `INV-INV-001` の Doctor 診断がすべて PASS すること。
- **Post Incident Actions:**
  - RCA の実施。同一 `failureClass` が再発する場合は ADR-022 の分類定義またはワーカー実装を見直す。

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
| Outbox / DLQ | DLQ Depth / Oldest Unprocessed Event Age |
| RBAC Policy Engine | Fail-Closed 由来 DENY 率 (`deniedBy = FailClosedGuarantee`) |
| Accounting Export | `EXPORT_FAILED` 滞留件数 |

### Alert Policies (失敗検知のしきい値)
以下は ADR-022 2.5（失敗の可視化義務）に対応する必須アラートです。いずれも「人がダッシュボードを見に行く」ことを検知手段としないことを原則とします。

| Alert | 条件 | 対応 Runbook |
|---|---|---|
| Webhook Signature Failure | 検証失敗 5 件 / 5 分 | IR-001 |
| Webhook 5xx Rate | 5xx 率 > 1% / 5 分 | IR-001 |
| Accounting Export Stall | `EXPORT_FAILED` ≥ 1 件が 30 分継続 | IR-002 |
| DLQ Depth | DLQ 件数 ≥ 1 | IR-003 |
| Outbox Lag | 未処理イベント最古齢 > 15 分 | IR-003 |
| Fail-Closed DENY Spike | 障害由来 DENY 率 > 0.5% / 5 分 | IR-001 / 認可基盤調査 |
| Reconciliation Mismatch | 日次対査の差異 ≥ 1 件 | MO-002 |

### RM-001: 定期メンテナンスタスク一覧
| 頻度 | タスク内容 | Success Criteria |
|---|---|---|
| **Daily** | Cloud Error Reporting の確認 | `Error Count (未処理 5xx) = 0` |
| **Daily** | DLQ / Outbox 滞留の確認（IR-003） | DLQ 件数 = 0、未処理イベント最古齢 < 5 分 |
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
