# Architecture Health Score & Go-Live Decision

## 1. Scorecard (100点満点評価)

| Category | Score / 10 | Comment & Evidence |
|---|---|---|
| **Architecture (Domain Design)** | 10 | [Phase 6~10] Hexagonal Architecture, Aggregate Boundary の分離が完璧。不変な Ledger/Journal モデルが確立済 (ADR-001~015)。 |
| **Security** | 10 | [Phase 11.1] Firebase Security Rules、テナント分離 (ADR-016)、Custom Claims、Webhook 署名検証 (ADR-020) が堅牢に実装・UAT合格済。 |
| **Reliability** | 6 | 冪等性制御は優秀だが、Firebase Emulator を用いた完全な E2E カオス/結合テスト（Race Condition 検証）が実施されていないため減点。 |
| **Operations** | 5 | デプロイフローは存在（Vercel）するが、Secret Rotation 自動化、IaC（Infrastructure as Code）化が未熟。 |
| **Compliance** | 10 | [ADR-019, 021] 監査トレールと RFC8785 Canonical Hash による会計データの完全性保存要件を完璧に満たしている。 |
| **Performance** | 8 | Firestore Composite Index による N+1 回避は良好。ただし、月次締め等の大規模バッチのトランザクション限界テストが未了。 |
| **Observability** | 2 | 🚨 **Critical Issue**: Correlation ID（相関ID）と構造化ログのパイプラインが未実装。障害時の分散トレースが不可能。失敗の伝播規律は ADR-022 で仕様化済 (RSK-010) だが、アラート設定証跡は未提示。 |
| **Disaster Recovery** | 5 | PITR への依存は宣言されているが、復旧手順のドリル（訓練）が実証されていない。 |
| **Maintainability** | 9 | `svelte-check` 0 error (若干の a11y 警告あり)、TypeScript の厳格な型推論、ドキュメントの網羅性は非常に高く優秀。 |
| **Production Readiness** | 5 | E2E の最終結合とログ監視基盤の欠如が、ビジネスリスクを押し上げている。 |

**総合スコア: 70 / 100**

- **Architecture Maturity Level**: Level 4 (Advanced Domain Driven)
- **Enterprise Readiness Level**: Level 3 (Needs Observability & Resilience Testing)

---

## 2. Review Findings (改善提案)
*※ Architecture Freeze 制約に違反しない範囲でのインフラ・運用層の改善案*

1. **[Observability] `CloudLogger` の導入と Correlation ID パスの確立**
   - 既存のドメイン集約メソッドの引数は変更せず、インフラ層（Web Framework / Middleware レイヤ）にて `AsyncLocalStorage` を活用し、HTTPリクエストごとに一意の `traceId` を割り振る。
   - `console.log` をラップし、GCP 互換の JSON 形式（`severity`, `message`, `traceId`）で出力する。
2. **[Reliability] Firebase Emulator E2E テストの拡充**
   - ドメインモデルは一切変更せず、`scripts/run-e2e-emulator.ts` を新規作成し、Attendance から Accounting Export までの通し検証、および Duplicate Webhook 等の異常系アサーションを実装する。
3. **[Operations] Secret ローテーション手順の自動化**
   - Google Cloud Secret Manager (または Firebase App Check / Secrets) を活用した動的読込へ移行する。
4. **[Reliability] Silent Failure の再発防止 (ADR-022 の強制検査)**
   - ドメインモデルは変更せず、受信端点と非同期ワーカーに対して「未処理のまま 2xx を返さない」「失敗イベントを `PROCESSED` にしない」ことを Doctor 診断ルール (`DOC-REL-001`) および Emulator 異常系シナリオで強制検査する。
   - Lint レベルでは空 `catch` および `cause` を破棄した再スローを検出するルールを CI ゲートに追加する。

---

## 3. Executive Decision

### 🟡 APPROVED WITH CONDITIONS（条件付き承認）

現在のアーキテクチャの根幹（ドメインロジック、不変データモデル、セキュリティ境界）は非常に完成度が高く、金融トランザクションを安全に処理する能力を十分に備えています。設計（Architecture Freeze）は完全に妥当であり、これ以上の設計変更は不要です。

しかしながら、本番稼働において不可欠となる **「可観測性（Correlation IDと構造化ログ）」** および **「E2E結合レベルでの回復力検証」** が欠如しています。

**Go-Live 前に必須となる条件 (Blockers):**
- 構造化ログ（CloudLogger）の実装および相関IDの付与。
- Emulator を用いた End-to-End のフルサイクル結合テスト（正常系＋冪等性防御）のパス。
- Cloud Functions のリトライ有効化設定。
- ADR-022 に定める必須アラート（DLQ 件数 / Outbox 滞留 / `EXPORT_FAILED` 滞留 / 署名検証失敗）の設定証跡。

上記の「運用・検証基盤の追加（機能実装ではなくインフラ整備）」が完了した時点で、スコアは 90 点を超え、**✅ APPROVED FOR PRODUCTION** へステータスが更新されます。直ちにこれらのインフラタスクに着手することを強く推奨します。
