# ASAHI Coach App Production Readiness Risk Register v1.0

本ドキュメントは、ASAHI Coach App の本番環境における潜在的リスクを列挙し、重大度（Severity）別に分類したものです。すべてのリスクは、Phase 6〜11.3 までの実証結果（提出された ADR および実装コード等）を根拠としています。

*※ 本書において Trace ID（HTTPリクエスト単位）と Correlation ID（業務トランザクション単位）を区別する場合は、今後の運用マニュアル等でそれぞれの用途を明確に定義することを推奨する。*

## 1. Risk Management Roles (リスク管理体制)

本レジスタの運用における各ロールの責任範囲は以下の通りです。

| Role | Responsibility |
|---|---|
| **Owner** | 各リスクに対する Mitigation (対策) の実施、および証跡の提出責任。 |
| **Reviewer** | 提出された証跡の妥当性評価、および対策完了（Mitigated/Closed）へのステータス変更承認。 |
| **Approver** | Go-Live (商用稼働) の最終判定および、リスク受容（Accepted）の最終承認。 |

### Status 遷移ルール

| Status | 意味 |
|---|---|
| **Open** | 未対応 |
| **In Progress** | 対策実施中 |
| **Mitigated** | 対策済・効果確認待ち |
| **Closed** | Reviewer承認済 |
| **Accepted** | リスク受容済 (Approverの承認を要する) |

## 2. Open Risk一覧 (Risk Matrix)

| Risk ID | Status | Category | Risk Description | Severity | Probability | Impact | Mitigation (対策) | Owner | Due Date |
|---|---|---|---|---|---|---|---|---|---|
| **RSK-001** | Open | Observability | **[ADR-021/Phase 11.3] 決済追跡用 Correlation ID の実装証跡未確認**<br>提出された成果物には、Webhook受信からJournalEntry生成までを追跡するCorrelation IDの実装または設定を裏付ける証跡が含まれていなかった。このため、障害時のトランザクション特定が困難となるリスクがある。 | **Critical** | High | High | `CloudLogger` と `AsyncLocalStorage` を導入し、エッジで生成した Trace ID/Correlation ID をすべてのログに自動付与する設定証跡を提示する。 | SRE | Pre Go-Live |
| **RSK-002** | Open | Reliability | **[Phase 11.3] カオス/E2E テストの実証記録未提示**<br>提出成果物には、Firebase Emulator を用いた競合状態（Race Condition）やネットワーク切断（Timeout Recovery）の E2E 検証が完了したことを裏付ける証跡が含まれていなかった。Firestore トランザクションロックの限界値が不明。 | **High** | Medium | High | Go-Live 前に Emulator Suite を用いた並行リクエスト処理（Concurrent Payments）の負荷テストシナリオを実装・パスさせ、実証記録を提示する。 | QA/SRE | Pre Go-Live |
| **RSK-003** | Open | Operational | **[Phase 11.1] Secret Rotation 自動化の実装証跡未確認**<br>提出物には、Stripe/GMO/freee の API Key および Webhook Secret のローテーション手順や IaC 設定を裏付ける証跡が含まれていなかった。キー漏洩時の無停止更新が不可となるリスクがある。 | **High** | Low | Critical | Google Cloud Secret Manager への移行と、ダウンタイムなしでのキー再読込機構（Cloud Functions の再デプロイ自動化）を構築し、構成証跡を提示する。 | Security | Pre Go-Live |
| **RSK-004** | Open | Disaster Recovery | **[ADR-019] Firestore バックアップからのリストア実証記録未提示**<br>Firestore Point-in-Time Recovery または Backup Policy の有効化状態について宣言はあるが、実際の復元成功を裏付けるリストアドリル証跡が含まれていなかった。 | **High** | Low | Critical | 非本番環境を用いたリストアドリルを月次で実施する手順を Operations Manual に追加し、実証記録を提示する。 | SRE | Pre Go-Live |
| **RSK-005** | Open | Performance | **[ADR-011] 月次締め (Month End Close) 時の書込負荷試験証跡未提示**<br>提出物には、月末に全テナントの Invoice が一斉に生成・確定される際のトランザクション限界（約500/秒）を想定した負荷試験の実施を裏付ける証跡が含まれていなかった。 | **Medium** | High | Medium | 月次締め処理をテナントごと・チャンクごとに分散（Sleep挿入など）させる非同期処理へ移行し、1000 Families 規模の実証記録を提示する。 | Backend | Post Go-Live |
| **RSK-006** | Open | Compliance | **[ADR-019] Cold Archive 自動化の実装証跡未提示**<br>提出物には、25ヶ月経過後の財務データの Storage（Read-only Archive）への退避自動化設定を裏付ける証跡が含まれていなかった。Firestore ストレージコストの増大および監査時エクスポートの遅延リスクがある。 | **Low** | High | Low | 基準日を過ぎたデータを自動的にエクスポートして Storage に保存・Firestore から削除する Lifecycle ロジックを実装し、証跡を提示する。 | SRE | Post Go-Live |
| **RSK-007** | Open | Compliance / Audit | **[ADR-001/ADR-004] Audit Trail Integrity (監査証跡の完全性検証手順)の実証証跡未確認**<br>Immutable Ledger および RBAC Decision Log の長期監査証跡保持について、改ざん検知および保存期間経過後の検証手順を裏付ける実証証跡が含まれていなかった。 | **High** | Low | Critical | Ledger Entry、AuthorizationDecision、Emergency Override Log に対する Hash Chain または Canonical JSON Hash 検証プロセスを Operations Manual に追加し、監査復元テストを実施する。 | Security | Pre Go-Live |
| **RSK-008** | Open | Distributed Transaction | **[ADR-006/ADR-007] Eventual Consistency Failure (結果整合性障害)からの復旧運用証跡未確認**<br>Outbox/Saga Compensation により非同期整合性を保証しているが、外部決済・会計システム障害時の補償処理再実行およびDead Letter Queueの運用を裏付ける証跡が含まれていなかった。 | **High** | Medium | High | Outbox Event状態(State Machine)、Retry Policy、DLQ監視、Manual Replay手順をOperations Manual化し、障害復旧テストを実施する。 | SRE | Pre Go-Live |
| **RSK-009** | Open | Security | **[ADR-004/ADR-005/ADR-009] RBAC Policy Drift (ポリシー乖離)検証手順の実証証跡未確認**<br>Permission Catalog SSOT と Role Policy の変更時、既存AuthorizationDecisionとの互換性検証手順を裏付ける証跡が含まれていなかった。 | **Medium** | Medium | High | Permission Catalog Versioning、Migration Rule、Regression Test SuiteをCI/CD Gateへ追加する。 | Security | Post Go-Live |

## 3. Risk Definitions
- **Critical:** Go-Live 前に必ず解消しなければならない致命的リスク。データ消失、あるいは法的/財務的証跡が追跡不能になるもの。
- **High:** Go-Live 判定を保留とする強い理由となるリスク。発生時のビジネスインパクトが甚大。
- **Medium:** Go-Live は可能だが、運用中に高い確率で顕在化し、運用コストを増大させるリスク。
- **Low:** Go-Live に影響はないが、長期的（1〜3年スパン）に技術的負債やコスト増大につながるリスク。

## 4. Go-Live Acceptance Criteria (Go-Live判定基準)

本レジスタに記載されたリスクに対し、以下の基準で Go-Live 可否（Gate）を設定する。

### Evidence of Completion (対策完了の証跡)
対策の完了（Status = Mitigated / Closed）は、以下のいずれかまたは組み合わせが提出され、Reviewer によって確認されることで証明される。
- ADR の更新およびコミット
- CI/CD パイプラインの実行ログ
- 自動テスト（Regression Test Suite）のパス結果
- Emulator による E2E テスト・障害テストの実行結果
- Cloud Monitoring / Error Reporting 等のダッシュボード設定画面のキャプチャまたはエクスポート
- Operations Manual の手順更新
- 監査ログの出力実例
- リストアドリルの実施記録

### 🔴 Blocker（Go-Live不可 - 必須解消事項）
商用トラフィックを受け入れる前に、必ず証跡を提示して解消しなければならない事項。
- **RSK-001** Correlation ID
- **RSK-003** Secret Rotation
- **RSK-004** Restore Drill

### 🟡 Conditional Approval（条件付き承認 - 運用カバー等を条件にGo-Live可）
Go-Live 判定までに、少なくとも一時的な運用回避策や実証検証の完了証跡を要する事項。
- **RSK-002** Chaos/E2E
- **RSK-007** Audit Integrity
- **RSK-008** Saga Recovery

### 🟢 Post Go-Live Hardening（稼働後強化事項）
Go-Live をブロックしないが、サービススケールや将来的なコンプライアンス維持のために期日を定めて解消すべき事項。
- **RSK-005** Month Close Load
- **RSK-006** Cold Archive
- **RSK-009** RBAC Drift

## 5. Evidence Link一覧

各リスクの評価根拠となった提出物および関連ファイルの一覧です。（実装ファイルやテストコード等は、今回提出物として確認できた範囲で記載）

| Evidence / File | Related Risks |
|---|---|
| [ADR-021: Traceability](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-001 |
| [Phase 11.3 UAT Reports](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/walkthrough.md) | RSK-001, RSK-002, RSK-008 |
| `firebase.json` / `.env.vercel.prod` | RSK-003 |
| [ADR-019: Data Retention](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-004, RSK-006 |
| [ADR-011: Month End Close](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-005 |
| [ADR-001: Immutable Ledger](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-007 |
| [ADR-004: Canonical JSON Hash](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-007, RSK-009 |
| [ADR-006 & 007: Outbox / Saga](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-008 |
| [ADR-005: Permission Catalog SSOT](file:///C:/Users/kawai/.gemini/antigravity/brain/f2d96dc4-9b44-4931-bc19-8f679cf8075a/architecture_decision_records.md) | RSK-009 |
| `run-security-rules-uat.ts` (Security UAT) | RSK-009 |

---

## 6. Review Basis & Conclusion

本レビューは、提出された ADR、実装コードおよび関連成果物のみを対象として実施した。

レビュー対象に含まれないクラウド設定、運用手順、監視設定、インフラ構成については評価対象外とし、確認できなかった事項は「未提示」「未確認」として記載した。

本書で示した改善事項は、提出物ベースのレビュー品質および運用・監査性の向上を目的とした推奨事項であり、実装の有無を断定するものではない。

---

## 7. Approval

| Role | Name | Date | Result |
|---|---|---|---|
| Technical Reviewer | — | — | Approved / Rework |
| Architecture Reviewer | — | — | Approved / Rework |
| Go-Live Approver | — | — | Approved / Rejected |
