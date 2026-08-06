# Architecture Decision Records (ADR-001 ~ ADR-007)

本ドキュメントは、「スポーツクラブ運営システム（ASAHI Coach App）」における主要なアーキテクチャ設計判断（ADR: Architecture Decision Records）を文書化したものです。Phase 6（Wallet）から Phase 9（RBAC Enterprise Core Frozen）、および Phase 10（Accounting & Payment Integration）を見据え、数年間にわたり長期保守可能な設計不変事項とガバナンス規律を定義します。

---

## 【ADR Index / Master Summary】

| ADR ID | タイトル | ステータス | 決定日 | 対象領域 | 概要 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **ADR-001** | **WalletにおけるイミュータブルLedger方式の採用** | Accepted (Frozen) | 2026-08-01 | Wallet | 追記型元帳による残高計算と Saga Compensation による逆仕訳原則 |
| **ADR-002** | **InvoiceにおけるPricing Snapshot方式の採用** | Accepted (Frozen) | 2026-08-01 | Invoice | 発行時点の料金体系・税率の不変スナップショット保持と経理不変条件 |
| **ADR-003** | **Attendanceを業務ライフサイクルの Hub Domain とする設計** | Accepted (Frozen) | 2026-08-01 | Attendance | 出欠を起点とした配車・活動費補助・請求相殺の単方向依存統制 |
| **ADR-004** | **RBACにおける純粋関数ポリシーおよび Fail-Closed Guarantee** | Accepted (Frozen) | 2026-08-01 | RBAC | 例外・タイムアウト・競合発生時の「安全側拒否 (DENY)」不変方針 |
| **ADR-005** | **Permission Catalogの Single Source of Truth (SSOT) 化** | Accepted (Frozen) | 2026-08-01 | RBAC | 全22権限のカタログ管理および SHA-256 バージョン検証による権限スプロール防止 |
| **ADR-006** | **Domain Event Integration (Saga / Outbox パターン方針)** | Accepted | 2026-08-01 | Integration | 5大ドメイン間のイベント駆動連携における順序保証と非同期補償トランザクション |
| **ADR-007** | **External System Boundary (決済・会計外部連携境界)** | Accepted | 2026-08-01 | Accounting | Stripe / freee / Wallet / Invoice における責務分離と真実の境界 (SSOT Boundary) |
| **ADR-008** | **Emergency Override Protocol (Break Glass Access)** | Accepted (Frozen) | 2026-08-01 | Governance | 危機発生時の 4段階非常事態ロック・Two Person Rule および監査必須解除プロトコル |
| **ADR-009** | **Freeze RBAC Core Architecture (v1.0.0)** | Accepted (Frozen) | 2026-08-01 | RBAC | 権限カタログのセマンティックバージョン管理 (`v1.0.0`) およびコア仕様変更の ADR 必須化 |
| **ADR-010** | **External Accounting Boundary (決済・会計イベント駆動境界)** | Accepted | 2026-08-01 | Accounting | RBAC・ビジネス・外部決済会計間の直呼出し禁止とイベント駆動アダプター原則 |
| **ADR-011** | **Ledger Closing Policy (会計締め・期間ロックポリシー)** | Accepted | 2026-08-01 | Accounting | 月次締め (`MONTH END CLOSE`) 後の過去元帳書変え禁止と補正仕訳 (`Correction Entry`) 原則 |
| **ADR-012** | **Payment Lifecycle State Machine (決済状態遷移モデル)** | Accepted | 2026-08-01 | Payment | 決済状態遷移 (`DRAFT -> ISSUED -> PAID -> SETTLED`) および非同期イベント変更原則 |
| **ADR-013** | **Reconciliation Policy (対査・入金消込・整合性ポリシー)** | Accepted (Frozen) | 2026-08-01 | Accounting | 内部 Invoice・決済プロバイダ・freee 法定元帳間の日次/月次自動対査と乖離検知 |
| **ADR-014** | **Payment Provider Adapter Boundary (外部決済プロバイダ中立隔離原則)** | Accepted (Frozen) | 2026-08-01 | Payment | Stripe / GMOあおぞらネット銀行 API の完全隔離と中立決済事実への正規化 |
| **ADR-015** | **Accounting Export Boundary & Statutory GL Integration Policy** | Accepted (Frozen) | 2026-08-01 | Accounting | freee を計算エンジンにしない分離・汚染ゼロ原則と 4ステップ月次締め同期 |
| **ADR-016** | **Tenant Boundary Enforcement Policy (マルチテナント隔離・所属不変原則)** | Accepted (Frozen) | 2026-08-01 | Governance | organizationId 必須化および変更・他テナント参照・所属異動の厳格遮断 |
| **ADR-017** | **Server Authority Boundary (クライアント書込制限と不変元帳サーバー専用更新原則)** | Accepted (Frozen) | 2026-08-01 | Security | Wallet/Invoice/Accounting/Auditのクライアント更新禁止とサーバー権限境界 |
| **ADR-018** | **Custom Claims Refresh Consistency (権限・テナント変更時のSSOT同期プロトコル)** | Accepted (Frozen) | 2026-08-01 | Security | Role/Tenant変更時の4ステップ固定同期(DB更新→監査→Claims→Refresh要求) |
| **ADR-019** | **Financial Data Retention & Archive Policy (金融・監査データ保存とアーカイブ原則)** | Accepted (Frozen) | 2026-08-01 | Governance | Active(0-24ヶ月)→Archive(25ヶ月-7年)→Legal(7年以上)の不変保存・物理削除禁止原則 |
| **ADR-020** | **External Integration Boundary Security (外部決済API非信頼ゾーン隔離原則)** | Accepted (Frozen) | 2026-08-01 | Security | 外部決済APIを非信頼ゾーンとし、Verifier→Mapper→DomainEvent経由のみドメイン変更許可 |
| **ADR-021** | **Payment Evidence Preservation (決済証跡・Webhook検証結果保存原則)** | Accepted (Frozen) | 2026-08-01 | Audit | providerEventId/Timestamp/検証結果/ハッシュを不変保存する決済監査証跡原則 |

---

## ADR-001: WalletにおけるイミュータブルLedger方式の採用

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
スポーツクラブ運営におけるウォレット（活動費補助金・事前預り金）管理では、資金移動や残高計算において競合や監査不能状態を防ぐ確実な方式設計が必要でした。

### 2. Alternatives Considered (検討された代替案)
- **案A: Balance Update 方式 (直接変更型)**:
  - ユーザーテーブルの `balance` カラムを `UPDATE balance = balance + amount` で直接変更。
  - *メリット*: 実装が容易で参照クエリが高速。
  - *デメリット*: 並行更新時に TOCTOU 競合が発生しやすく、過去の「誰がいつなぜポイントを付与したか」の完全な証跡が残らない。
- **案B: Immutable Ledger 方式 (追記型元帳型)**:
  - 変更をすべて不変トランザクション行 (`WalletLedgerEntry`) として追加し、残高は元帳の集計によって導出する。
  - *メリット*: 100%の監査追跡性、べき等キー (`transactionKey`) による二重計上防止機能との高い親和性。

### 3. Decision (決定事項)
**案B: Immutable Ledger 方式**を採用します。残高変更はインサートのみとし、いかなる `UPDATE` / `DELETE` も禁止します。

### 4. Assumptions (前提条件)
- コイン・補助金は金融トランザクションと同様の監査基準を要します。
- 一括消込における残高不変条件 (`Rule-WAL-001` / `INV-WAL-001`) を常に満たします。

### 5. Constraints (制約事項)
- 物理削除（`DELETE FROM wallet_ledgers`）をデータベースのロール・アプリ双方で完全に禁止。
- 取消や減額は、必ず負値または逆仕訳トランザクション (`Reversal`) を発行して補償。

### 6. Consequences (効果と影響)
- **Pros**: 二重相殺・二重付与をコンパイル時および DB レベルで完全に防止可能。
- **Cons**: レコード件数増加に伴い、数年後には月次アーカイバスナップショットによるクエリ最適化の検討が必要。

---

## ADR-002: InvoiceにおけるPricing Snapshot方式の採用

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
月次会費バッチ発行において、基本会費、兄弟割引、休会維持費、あるいは国の消費税率は年々変更される可能性があります。発行済み請求書が現在価格マスターを参照する設計では、改定後に過去請求書の計算額が変わってしまう致命的欠陥が生じます。

### 2. Alternatives Considered (検討された代替案)
- **案A: Master Reference 方式 (参照型)**:
  - `invoice_items.price_master_id` を持ち、価格・税率は JOIN で取得。
  - *デメリット*: マスター改定時に歴史的会計レコードが破壊される。
- **案B: Pricing Snapshot 方式 (スナップショット型)**:
  - 発行瞬間の単価・割引額・税率・項目名を `InvoiceItemSnapshot[]` に完全複写して保存。

### 3. Decision (決定事項)
**案B: Pricing Snapshot 方式**を採用します。発行済み請求書は不変オブジェクトとして保存します。

### 4. Assumptions (前提条件)
- 発行後の請求書における計算式（合計＝税抜＋消費税－割引－ウォレット相殺）は `Rule-INV-007` として恒久不変であること。

### 5. Constraints (制約事項)
- 一度発行された請求書の価格・明細カラムの上書き (`UPDATE`) を禁止。
- 変更・返金時はステータスを `VOID` / `REFUND` へ変更し、新しい訂正請求書を発行すること。

### 6. Consequences (効果と影響)
- **Pros**: 税率改定やクラブ会費規約変更が起きても、過去年度の税理士監査や会計検査に100%耐えうる正確性を維持。
- **Cons**: 明細テキストの複製によりストレージ消費が増加するが、現代の DB では無視できるコスト。

---

## ADR-003: Attendanceを業務ライフサイクルの Hub Domain とする設計

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
スポーツクラブ運営においては、「出欠・配車」「活動費ウォレット」「月次請求」が複雑に絡み合います。複数ドメインが互いに直接書き込み合う双方向依存を許すと、デッドロックや循環参照が生じます。

### 2. Alternatives Considered (検討された代替案)
- **案A: Mesh Dependency (メッシュ相互参照)**:
  - Wallet が配車完了を更新し、出欠が Invoice を更新する等、各ユースケースで自由に呼び出す。
- **案B: Attendance Hub Pattern (出欠起点・単方向フロー)**:
  - 現場の「出欠」を業務の唯一のハブドメインとし、依存方向を一方向に統一する。

### 3. Decision (決定事項)
**案B: Attendance Hub Pattern** を採用します。以下の単方向フローを不変則とします。

```
Attendance (出欠確定)
     │
     ▼
Ride (遠征配車確定)
     │
     ▼
Wallet (活動費補助 2,000円 自動クレジット)
     │
     ▼
Invoice (月会費請求書発行 ＆ ウォレット自動相殺)
```

### 4. Assumptions (前提条件)
- **Attendance は必ず Ride より先に確定する**（出欠回答のない選手に配車は生成されない）。
- **Ride の完了なくして Wallet 補助金クレジットは発生しない**。

### 5. Constraints (制約事項)
- 下流ドメイン（Wallet / Invoice）から上流ドメイン（Attendance / Ride）のドメインエンティティを直接修正することを禁止。

### 6. Consequences (効果と影響)
- **Pros**: ドメイン境界が非常にクリアになり、各チームやフェーズごとの独立開発・並行テスト（UAT）が極めて容易になる。
- **Cons**: 業務フローを遡る訂正（例: 配車完了後の出欠キャンセル）には、Saga 補償トランザクション（逆仕訳）を発行する必要がある。

---

## ADR-004: RBACにおける純粋関数ポリシーおよび Fail-Closed Guarantee

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
エンタープライズクラブ運営の認可（RBAC / ABAC / SoD / 一時委譲）において、認可エンジンが高負荷になった際や障害発生時にセキュリティホール（未認証アクセス許可）が生じることを防ぐ必要があります。

### 2. Alternatives Considered (検討された代替案)
- **案A: DB-Driven Policy Evaluation (DB問い合わせ型評価)**:
  - 評価のたびに DB にアクセスして組織やロールを引く。
  - *デメリット*: 大量アクセス時に DB ボトルネックとなり、タイムアウト時にエラー処理が曖昧になりやすい。
- **案B: Pure-Function Evaluators ＋ Fail-Closed Guarantee (純粋関数型＋安全側拒否)**:
  - 評価コンテキストを事前に用意し、全8層のポリシー評価器をサイドエフェクトなしの純粋関数として実行する。

### 3. Decision (決定事項)
**案B: Pure-Function Evaluators ＋ Fail-Closed Guarantee** を採用します。いかなる評価中例外・タイムアウト・ETag競合も自動で `DENY` (`deniedBy: 'FailClosedGuarantee'`) へ安全退避させます。

### 4. Assumptions (前提条件)
- 認可決定結果 `AuthorizationDecision` は不変（`Object.freeze()`）であり、RFC 8785 Canonical JSON を用いた SHA-256 署名を保持すること。

### 5. Constraints (制約事項)
- **ポリシー評価器 (`evaluate()`) 内での副作用・DBアクセス・HTTP通信・外部ネットワーク呼び出しを一切禁止**。
- **ポリシー内部でのシステム時計 (`Date.now()`) の直接取得を禁止**（評価日時 `now` は必ず実行コンテキストから注入）。

### 6. Consequences (効果と影響)
- **Pros**: **実測値 21,490 req/sec、平均レイテンシ 46μs** という圧倒的高速評価と、いかなる障害発生時もセキュアなフェイルセーフを確立。
- **Cons**: 認可サービス（`RbacService`）において、必要なコンテキストを LRU キャッシュまたは事前クエリで構築する責務を負う。

---

## ADR-005: Permission Catalogの Single Source of Truth (SSOT) 化

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
コードベース全域で `'invoice:pay'` や `'invoice:payment'` のように似た名前の権限文字列が分散ハードコードされると、タイポによる権限漏洩やデッドコード（権限スプロール現象）が生じます。

### 2. Alternatives Considered (検討された代替案)
- **案A: Free-Form String Permissions (自由記述文字列型)**:
  - 文字列であれば任意の権限キーを与えられる。
- **案B: Centralized Permission Catalog (SSOT 厳格カタログ型)**:
  - 全22権限を `PERMISSION_CATALOG` として型マッピングし、未登録権限の使用をコンパイル時および Doctor 診断で遮断。

### 3. Decision (決定事項)
**案B: Centralized Permission Catalog** を採用し、唯一の権限定義情報源（SSOT）とします。

### 4. Assumptions (前提条件)
- すべての標準ロール（`admin`, `coach`, `parent`, `player`, `treasurer`）およびカスタム許可/拒否、一時委譲（`DelegationGrant`）は SSOT カタログ内のキーを参照する。

### 5. Constraints (制約事項)
- レジストリの初期化後改変（プラグインの動的差替・優先度上書き）を `PolicyRegistry.freeze()` によって実行時に禁止。

### 6. Consequences (効果と影響)
- **Pros**: 廃止・存在しない権限による設定ミスがゼロになり、安全なガバナンスを強制可能。
- **Cons**: 権限追加時に標準的なリリース手続きを踏む必要がある（下記 Future Considerations 参照）。

### 7. Future Considerations (将来運用ルール・権限追加ワークフロー)
新たな業務要件により権限キーを追加する際は、以下の標準手順を義務付けます。
```
[1. PERMISSION_CATALOG への定義追加 (RbacModel.ts)]
                         │
                         ▼
[2. Doctor 診断ルールの更新確認 (DOC-RBAC-003, DOC-RBAC-018)]
                         │
                         ▼
[3. UAT シナリオテストの拡充 (RbacUAT.ts)]
                         │
                         ▼
[4. Schema SHA-256 Hash の更新・バージョン刻印 (2026.08-v2 等)]
```

---

## ADR-006: Domain Event Integration (Saga / Outbox パターン方針)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
Phase 10（会計・決済）以降、Attendance, Ride, Wallet, Invoice, Accounting 間の連携において、同期的な REST/関数呼び出しのみに依存すると、一時的な DB ロックや決済ゲートウェイ遅延により連鎖障害を引き起こします。

### 2. Alternatives Considered (検討された代替案)
- **案A: Synchronous Cascade Call (同期連鎖呼び出し)**:
  - 配車完了メソッドから直接 Wallet の Ledger 作成と Invoice の相殺メソッドを同期呼び出し。
  - *デメリット*: どこか一つがロックまたは失敗すると全体がロールバックされ、応答性が悪化する。
- **案B: Transactional Outbox ＋ Saga Compensation Pattern (非同期イベント＆補償型)**:
  - メインエンティティ変更時に同じ DB トランザクション内でイベント（`RideCompletedEvent` 等）を Outbox テーブルに書き込み、ワーカーが非同期で下流に伝播。例外時は Saga 補償トランザクションを発行。

### 3. Decision (決定事項)
**案B: Transactional Outbox ＋ Saga Compensation Pattern** を標準採用します。

### 4. Assumptions (前提条件)
- すべてのドメインイベントはべき等キー (`eventId` / `idempotencyKey`) を持ち、下流コンシューマでの重複処理（Rule-ATT-005, Rule-WAL-004）を防止します。

### 5. Constraints (制約事項)
- ドメイン境界をまたぐビジネス処理の反映は「結果整合性 (Eventual Consistency)」を許容する（ただし Wallet の Ledger 集計値そのものは強整合性）。

### 6. Consequences (効果と影響)
- **Pros**: 各ドメインサービスが自律的に稼働し、決済 API 一時障害時も再試行 (`Retry`) によりセルフヒーリングが可能。
- **Cons**: UI 画面上での「反映待ち時間」に対する通知設計（トースト通知や WebSocket 更新等）が必要となる。

---

## ADR-007: External System Boundary (決済・会計外部連携境界)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
Phase 10 において外部決済サービス（Stripe / GMO決済等）およびクラウド会計システム（freee / マネーフォワード等）を統合するにあたり、「どれがシステムとしてのお金の真実（Single Source of Truth）なのか」の責務境界を定義しないと、データ不整合が起きた際に改ざんや誤仕訳が生じます。

### 2. Alternatives Considered (検討された代替案)
- **案A: All-in-One Database SSOT (アプリ DB 全知全能型)**:
  - すべての残高・請求・決済ステータスの正とする真実を社内 DB に置く。
  - *デメリット*: クレジットカードチャージバックや税務申告において外部システムとの不整合が修正できない。
- **案B: Segregated Single Source of Truth Boundary (真実の責任分界点型)**:
  - 外部システムと内部システムの責任境界を明示的に分割し、各ドメインの「真実」を独立させる。

### 3. Decision (決定事項)
**案B: Segregated Single Source of Truth Boundary** を採用し、以下の厳格な真実境界 (SSOT Boundary) を定義します。

```
┌──────────────────────────────────────────────────────────────┐
│                  ASAHI Coach App (Internal)                  │
│                                                              │
│  [Wallet Domain]         [Invoice Domain]                    │
│   ・クラブ内活動費補助元帳   ・月次請求書スナップショット       │
│   ・家族間ポイント充当残高   ・兄弟割引・休会維持費計算結果      │
└───────────────┬──────────────────────────────┬───────────────┘
                │ Webhook (Idempotent)         │ API Sync
                ▼                              ▼
┌───────────────────────────────┐ ┌────────────────────────────┐
│      Stripe / GMO Payment     │ │    freee / クラウド会計     │
│  (Payment Execution SSOT)     │ │   (General Ledger SSOT)    │
│                               │ │                            │
│  ・実際のクレジットカード決済実行  │ │  ・法人の確定仕訳・財務諸表    │
│  ・入金確定 (Charge / Payout) │ │  ・税務申告用公式元帳          │
└───────────────────────────────┘ └────────────────────────────┘
```

- **Stripe / GMO Payment**: 実際の**カード決済執行・入金確定・チャージバックの唯一の真実 (SSOT)** とします。
- **freee / クラウド会計**: 法人の**確定税務仕訳・法定総勘定元帳の唯一の真実 (SSOT)** とします。
- **Wallet Domain**: クラブ内部の**活動費補助額・事前預り金の元帳残高の真実**とします。
- **Invoice Domain**: 会員家庭に対する**会費計算結果および請求書・領収書発行記録の真実**とします。

### 4. Assumptions (前提条件)
- Stripe Webhook（`payment_intent.succeeded` 等）の処理は必ず検証シグネチャ照合とべき等性チェック (`reconcileInvoice`) を介して実行されます。

### 5. Constraints (制約事項)
- アプリ側で直接カード情報を保持・保管することは PCI-DSS コンプライアンス上絶対に禁止。すべて Stripe の Customer / PaymentMethod トークンで管理します。
- freee への会計連携は、毎日/毎月の締め処理バッチ（またはイベント連携）により、内部 Invoice の確定情報を仕訳伝票として出力します。

### 6. Consequences (効果と影響)
- **Pros**: 「どこに本当の数字があるか」で悩むことがゼロになり、決済障害や会計監査に極めて強靭な構造となる。
- **Cons**: Stripe Webhook 欠落時のリトライおよび日次レコンシリエーション（対査）バッチの実装が推奨される。

---

## ADR-008: Emergency Override Protocol (Break Glass Access)

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
危機発生時やシステム非常災害時（ASTENO 思想における crisis operation）、単に書込みを停止する（`EmergencyPolicy -> write stop`）だけでなく、「誰がどのような安全規約のもとで非常事態ロックを解除し、緊急運用を行えるか」を明文化する必要があります。

### 2. Decision (決定事項)
4段階の **Emergency Override Protocol** を制定し、Break Glass（緊急突破）アクセスの解除資格と条件を以下に厳格固定します。

```
Level 0: Normal                (通常稼働)
Level 1: Emergency Lock        (全書込停止・保全モード)
Level 2: Emergency Operation   (縮退業務・緊急連絡モード)
Level 3: Break Glass Access    (非常事態強制オーバーライド権限)
```

- **Break Glass 対象ロール**: `理事長 (Chairman)`, `防災・セキュリティ責任者 (Security Officer)`, `システム統括管理者 (System Admin)` のみ。
- **解除・発動必須条件**:
  1. **Two-Person Rule (複数人承認)**: 単独発動を禁止し、上記権限者のうち**2名以上**の同時認証・署名を必須とする。
  2. **Mandatory Audit Logging**: 解除事由と操作ログは改ざん不能な RFC 8785 Canonical JSON ハッシュ付きで即時監査証跡として保管する。

### 3. Consequences (効果と影響)
- **Pros**: 「非常時に判断不能・手遅れになるリスク」を排除しつつ、単独権限乱用によるセキュリティ崩壊をコンプライアンス面から100%防止する。
- **Cons**: 災害対策訓練（DR 訓練）時に Break Glass 解除手順の演習と監査確認を行う運用負荷が生じる。

---

## ADR-009: Freeze RBAC Core Architecture (v1.0.0)

```
Status:        Accepted (Core Frozen)
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
RBAC 基盤が一般的なクラブアプリの枠を超え、自治体・学校連携や法人決済にも対応し得る「スポーツクラブ運営 OS の認可基盤 (ASAHI CLUB Operating System)」に昇華したため、不用意な機能拡張や設計改変によって既存の業務ライフサイクル（Attendance → Ride → Wallet → Invoice）が不整合に陥ることを防ぐ必要があります。

### 2. Decision (決定事項)
- **RBAC コアドメイン（Phase 9）の仕様・設計を正式に凍結（FROZEN）**します。
- **Permission Catalog Schema Versioning**:
  既存の SHA-256 ハッシュ（`7380c7925697e1ca`）に加え、セマンティックバージョン **`v1.0.0`** を付与し、不変条件とします。
  ```json
  {
    "schemaVersion": "rbac.permission.v1.0.0",
    "hash": "7380c7925697e1ca",
    "createdAt": "2026-08-01"
  }
  ```
- 以下の事項を変更する場合は、すべて **ADR の正式な承認手続き**を義務付けます。
  - `PERMISSION_CATALOG` のキー増減・仕様変更
  - 8層のポリシー優先度 (`Policy Priority`) または判定パイプライン順序の変更
  - `AuthorizationDecision` スキーマ・ハッシュ生成式 (`RFC 8785`) の変更
  - `Two-Person Rule` 等の非常事態プロトコル要件の変更

### 3. Consequences (効果と影響)
- **Pros**: 将来的に Phase 20 等で `v1 -> v2` のマイグレーションが必要となった場合でも、バージョンハッシュによる完全な後方互換性と変更追跡性を保証できる。
- **Cons**: RBAC に機能改修を加えたい場合でも、バグ修正およびセキュリティパッチを除き、ADR の文書化と審査が必須となる。

---

## ADR-010: External Accounting Boundary (決済・会計イベント駆動境界)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
Phase 10 における決済（Stripe / GMO決済）やクラウド会計（freee）との統合作業で、決済・会計の外部 API が直接 RBAC コアや業務エンティティを書き換えたり、逆に業務エンティティから直接外部 API を呼ぶ実装を行うと、サービス間の強結合やセキュリティバイパス、べき等性の欠落を招きます。

### 2. Decision (決定事項)
決済・会計連携は RBAC コアおよびビジネスドメインの内部に直接侵入させず、以下の3層境界とイベント駆動アダプター構造を標準とします。

```
                  ASAHI Coach App

        ┌───────────────────────────────┐
        │     Authorization Domain      │
        │      (Phase 9 RBAC Core)      │
        └───────────────┬───────────────┘
                        │
                  Permission Check
                        │
        ┌───────────────▼───────────────┐
        │        Business Domain        │
        │ Attendance / Ride / Wallet    │
        │ Invoice                       │
        └───────────────┬───────────────┘
                        │
             Authorized Accounting Event
              (PaymentRequested 等)
                        │
        ┌───────────────▼───────────────┐
        │      Integration Domain       │
        │ Stripe / GMO / freee Adapters │
        └───────────────────────────────┘
```

- **必須フロー (Event-Driven Flow)**:
  `Invoice Created` ──► `PaymentRequested Event` ──► `Payment Adapter` ──► `PaymentCompleted Event` ──► `Ledger Update` ──► `Audit Log`
- **厳格禁止事項 (Prohibitions)**:
  1. `❌ Invoice -> Stripe 直接呼出`
  2. `❌ Wallet -> freee 直接書込`
  3. `❌ Payment 結果による RBAC コア設定の直接変更`

### 3. Consequences (効果と影響)
- **Pros**: 決済サービスや会計クラウドが将来変更されても、`Wallet` / `Invoice` / `RBAC` の中核ドメインが影響を受けず、100% の自律性と耐障害性を維持できる。
- **Cons**: イベント仲介用アダプター層と結果整合性リトライハンドラの構築が必要となる。

---

## ADR-011: Ledger Closing Policy (会計締め・期間ロックポリシー)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
不変元帳 (`Immutable Ledger`) は永久追記型であるため、無制限に過去の記録が更新可能であると月次決算や確定会計との乖離が生じます。企業および地域スポーツ OS としての健全性を保つため、「会計締め (`Accounting Close`)」と期間ロックのルールを法規化する必要があります。

### 2. Decision (決定事項)
月次の締め処理サイクルおよび過去元帳に対する変更原則を以下に厳格化します。

```
Wallet Ledger

2026-08
OPEN ──► TRANSACTION RECORDING ──► MONTH END CLOSE ──► LOCKED
```

- **締め後変更の禁止**: 締め (`MONTH END CLOSE`) 完了後の期間 (`LOCKED`) に対する過去元帳エントリの修正・更新・削除をすべて禁止 (`❌ UPDATE/DELETE PROHIBITED`) とします。
- **補正仕訳原則**: ロック期間の訂正が必要な場合は、必ず新たな会計期間にて**補正追記エントリ (`Correction Entry` / 赤黒仕訳・逆仕訳)** を追加することによってのみ残高および記録を調整 (`✅ ALLOWED ONLY VIA CORRECTION ENTRY`) します。

### 3. Consequences (効果と影響)
- **Pros**: 過去の会計締め事実が確定され、公認会計士監査および税務署調査、自治体監査に対して100%の証拠能力を維持できる。
- **Cons**: 締め処理作業（締日バッチまたは管理者の締操作）を毎月の定型ワークフローに組み込む必要がある。

---

## ADR-012: Payment Lifecycle State Machine (決済状態遷移モデル)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
決済および請求に関する状態管理が曖昧であると、決済処理中 (`PAYMENT_PENDING`) の重複決済や、返却時の不整合が発生します。

### 2. Decision (決定事項)
請求および決済トランザクションの正規状態遷移マシーン (`Payment Lifecycle State Machine`) を以下に固定します。

```
[標準ライフサイクル]
DRAFT ──► ISSUED ──► PAYMENT_PENDING ──► PAID ──► SETTLED

[異常・キャンセル・返金ライフサイクル]
ISSUED ──► VOID
PAID   ──► REFUND_REQUESTED ──► REFUNDED
```

- **非同期イベント変更原則**: 状態遷移は同期的な API 直呼出しで行わず、すべて決済プロバイダからの非同期 Webhook および認証済イベント (`PaymentCompletedEvent`, `PaymentFailedEvent`, `RefundCompletedEvent`) によって駆動させます。

### 3. Consequences (効果と影響)
- **Pros**: すべての決済状態が唯一のステートマシンに統一され、重複請求・消込漏れ・不正返金をシステム構造レベルで排除できる。
- **Cons**: UI 側において非同期処理中のポーリングまたは WebSocket / SSE による状態更新反映 UI が必要となる。

---

## ADR-013: Reconciliation Policy (対査・入金消込・整合性ポリシー)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
クラブ内の請求結果 (`Invoice`)、外部決済事業者 (`Stripe` / `GMOあおぞら`) の実入金データ、および法人会計 (`freee`) の法定帳簿間で数字にズレ（未消込・手数料誤差・チャージバック）が生じた場合、それを早期検知・補正するポリシーが必要です。

### 2. Decision (決定事項)
3者間の日次・月次自動対査 (`Reconciliation`) ポリシーを制定し、金額の真実性を常に相互検証します。

```
[3-Way Reconciliation Triangle]
(1) Internal Invoice ◄── 対査 ──► (2) Stripe/GMO Settlement
        ▲                                ▲
        └─────────────── 対査 ────────────┘
                         ▼
             (3) freee GL Accounting
```

- **自動消込ルール**: 請求番号 (`invoiceId`) と決済インテント (`paymentIntentId`) を1対1または1対多で関連付け、不一致検出時は自動で `RECONCILIATION_DRIFT_ALERT` イベントを発行します。

### 3. Consequences (効果と影響)
- **Pros**: 数字の不整合を放置ゼロ化し、スポーツクラブ運営における金銭管理の透明性と監査耐性を最高水準に保つことができる。
- **Cons**: 対査バッチおよび差額（決済手数料・振込手数料など）の自動仕訳マッピングルールの設計が必要となる。

---

## ADR-014: Payment Provider Adapter Isolation (外部決済プロバイダー抽象化ポリシー)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
Phase 10.2 における決済機能実装で、Stripe SDK や GMOあおぞらネット銀行 SDK の型やメソッドをドメイン層（`PaymentModel` / `AccountingService` / `Invoice`）が直接参照・利用すると、決済事業者の仕様変更・APIバージョンアップ・マルチプロバイダー化の際に、凍結済みのコア会計モデルやビジネスロジックまで変更・汚染されるリスクが生じます。
また、Stripe Webhook 等が直接 `Invoice.status = 'PAID'` と書き換えるような設計は、ADR-010（外部会計境界）および ADR-011（元帳不変性原則）の甚大な違反となります。

### 2. Decision (決定事項)
決済プロバイダーとの統合にあたり、プロバイダー中立な抽象ポート（`PaymentProvider` Interface）を境界ポートとして定義し、外部 SDK の仕様はアダプター層内部でのみ解決すること（Hexagonal Architecture / Port-Adapter Isolation）を義務付けます。

```
                    Payment Domain (Core / Frozen)
                     │
                     │  Uses neutral interface
                     ▼
          <<Interface>> PaymentProvider
       ┌─────────────┴─────────────┐
       │                           │
       ▼                           ▼
StripePaymentAdapter       GmoAozoraPaymentAdapter
 (Stripe SDK Isolated)      (GMO API Isolated)
```

- **インターフェース定義方針 (`PaymentProvider` Interface)**:
  ```typescript
  interface PaymentProvider {
    createPaymentIntent(request: PaymentRequest): Promise<PaymentIntentRecord>;
    verifyWebhook(payload: string, signature: string): Promise<WebhookEvent>;
    refund(paymentId: string, amount: number, reason: string): Promise<RefundResult>;
  }
  ```
- **禁止事項 (Prohibitions)**:
  1. `❌ Stripe / GMO SDK の型を Payment Domain / Accounting Domain から import すること`
  2. `❌ Webhook ハンドラから直接 Invoice.status = PAID を更新すること`
- **正規決済完了イベントフロー (Mandatory Webhook Processing Flow)**:
  ```
  Stripe Webhook
        │ (verified & deduplicated by IdempotencyStore)
        ▼
  PaymentCompletedEvent
        │
        ▼
  Payment Domain (State Machine: PAID -> SETTLED)
        │
        ▼
  Invoice Settlement Process (Via Domain Event)
        │
        ▼
  Journal Entry (複式簿記仕訳作成: 1010 普通預金 / 5120 手数料 / 1110 売掛金)
  ```

- **Pros**: 決済プラットフォーム（Stripe / GMO / その他将来プロバイダ）の差分を 100% アダプター層に閉じ込めることができ、モックアダプター（`MockPaymentAdapter`）による全自動 UAT や単体テストが極めて容易になる。
- **Cons**: アダプター間での型変換層（DTO 変換）と、べき等性管理キーの保持機構（`IdempotencyStore`）の整備が必要となる。

---

## ADR-015: Accounting Export Boundary (法人会計クラウド外部同期境界ポリシー)

```
Status:        Accepted
Date:          2026-08-01
Owner:         Enterprise Architecture Team
Supersedes:    None
Superseded By: None
```

### 1. Context (背景と課題)
Phase 10.3 における freee（およびその他法人会計クラウド）連携において、freee API の仕様・勘定科目 ID・税区分コード・パートナータグを内部の複式簿記モデル (`JournalEntry` / `AccountingModel`) に混入させたり、請求発生や入金決済のたびに個別に freee API を直接呼び出して業務計算を行わせると、ASAHI Coach App の独立性と耐障害性が崩壊します。
また、月次締め処理 (`MONTH_END_CLOSE`) における不変条件（ADR-011）との同期順序が曖昧な場合、外部帳簿と内部帳簿の間で金額差異や不一致が生じる危険性があります。

### 2. Decision (決定事項)
freee 会計連携は「計算エンジン」として利用せず、**「確定した内部会計事実を法定帳簿へ一方向エクスポート・同期するアダプター」**と位置付けます。
中立なポートインターフェース `JournalExportPort` および変換マッパー `FreeeJournalMapper` を介して実装し、内部モデルへの freee API 依存の混入を固く禁じます。

```
              ASAHI Accounting Core (FROZEN)
          JournalEntry (id, account, amount, hash)
                           │
                           │  Neutral Export Port
                           ▼
          <<Interface>> JournalExportPort
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
     MockFreeeAccountingAdapter  FreeeAccountingAdapter
                │                     │
     (FreeeJournalMapper)     (FreeeJournalMapper)
                │                     │
                ▼                     ▼
         Mock Freee GL           freee GL API
```

- **インターフェース定義方針 (`JournalExportPort` Interface)**:
  ```typescript
  interface JournalExportPort {
    exportJournalEntries(request: ExportRequest): Promise<ExportResult>;
    syncMonthEndClose(period: AccountingPeriod): Promise<MonthEndCloseSyncResult>;
  }
  ```
- **厳格禁止事項 (Prohibitions)**:
  1. `❌ JournalEntry エンティティに freeeAccountItemId / freeePartnerId / freeeTaxCode 等の外部型・属性を持たせること`
  2. `❌ Invoice や Wallet の更新処理中に同期で freee API を直接呼び出すこと`
  3. `❌ ASAHI 側で仕訳確定前に freee 側で計算・締め処理を行うこと`
- **安全な月次締め同期シーケンス (Mandatory Month-End Close Sequence)**:
  ```
  OPEN (通常期間)
   │
   ▼
  Journal Export (当該月の全確定仕訳を freee GL へエクスポート・対査検証)
   │
   ▼
  freee GL Verified (監査対査 100% 一致確認)
   │
   ▼
  Month Close (AccountingService.closeAccountingPeriod 実行)
   │
   ▼
  LOCKED (ADR-011 元帳不変ロック確定)
  ```

### 3. Consequences (効果と影響)
- **Pros**: 法定会計クラウド側の仕様・API 変更・認証方式の変更から ASAHI Coach App のコア会計・認可ドメインを 100% 隔離でき、RFC 8785 ハッシュ証跡による改ざん検証が外部帳簿にまで拡張される。
- **Cons**: アダプター層において日本法定会計税区分およびマッピングテーブル (`FreeeJournalMapper`) の管理が必要になる。

---

## ADR-016: Tenant Boundary Enforcement Policy (マルチテナント隔離・所属不変原則)

```
Status:        Accepted (Frozen)
Date:          2026-08-01
Decision-Maker: Enterprise Architecture Team & Security Board
Domain:        Governance / Security
```

### 1. Context (背景と課題)
ASAHI Coach App は単独クラブ用の管理アプリから、複数の地域クラブ・学校法人・自治体が単一クラウドインフラ（SaaS）を共用する **地域スポーツ運営 OS** へと進化している。  
クラウドマルチテナント環境において最も重大なセキュリティ脅威は、権限の自己昇格（Admin 昇格）以上に、**「あるテナント（クラブ）のユーザーやデータを他テナントへ不正に移動・参照・書変えすること（クロスドメインテナント漏洩および移動）」** である。  

### 2. Decision (決定事項)
全ドメイン・全レコードに対して、以下の**テナント境界強制原則 (Tenant Boundary Enforcement Policy)** を適用し、Firestore Security Rules およびサーバー層で不変条件を強制する。

- **不変必須属性 (Mandatory Immutable Attributes)**:
  全ドメインの全レコード (`organizations`, `users`, `families`, `app-attendance-records`, `app-rides`, `app-activity-fee-transactions`, `invoices`, `journal-entries`, `audit-logs`) は、必ず以下の属性を保持しなければならない。
  ```typescript
  readonly organizationId: string; // Tenant ID (Immutable)
  readonly createdBy: string;      // User UID
  readonly createdAt: string;      // ISO Timestamp
  ```
- **禁止事項 (Prohibitions)**:
  1. `❌ organizationId の更新・変更（テナント移動の厳格禁止）`
  2. `❌ organizationId が異なる他テナントのドキュメントの read / write`
  3. `❌ request.auth.token.orgId と resource.data.organizationId が一致しないアクセス`
- **Firestore ルール検証例**:
  ```javascript
  allow update: if request.resource.data.organizationId == resource.data.organizationId;
  ```

---

## ADR-017: Server Authority Boundary (クライアント書込制限と不変元帳サーバー専用更新原則)

```
Status:        Accepted (Frozen)
Date:          2026-08-01
Decision-Maker: Enterprise Architecture Team & Security Board
Domain:        Security / Infrastructure
```

### 1. Context (背景と課題)
Phase 6〜10 において、Wallet Ledger、Invoice、Accounting Core、Audit Log 等の金融・監査ドメインモデルを構築したが、これらをクライアント層（ブラウザ / モバイル）から直接書き込み可能にした場合、フロントエンドの改変や API トークン悪用により不変元帳が破壊される恐れがある。

### 2. Decision (決定事項)
アクセス主体の責務に応じた **サーバー権限境界 (Server Authority Boundary)** を厳格に分離し、Firestore Security Rules に反映させる。

- **Client Write Allowed (クライアント直接書き込み許可領域)**:
  - `app-attendance-records`: 選手本人または保護者による出欠回答（締切前）、Coach による強制上書き (`attendance.override`)
  - `users` / `families`: ユーザー本人の許可されたプロフィール項目（氏名、電話番号、アイコン）、支払責任者による家族名称の変更
  - `app-rides`: Coach による配車計画作成、保護者・選手による乗車希望
- **Server Only (サーバー専用更新領域 — Client Write Strictly Forbidden)**:
  - `app-activity-fee-transactions` (活動費ウォレット元帳)
  - `invoices` (会費請求データ)
  - `journal-entries` / `accounting-periods` (法定・内部会計元帳)
  - `rbac-audit-logs` / `audit-logs` (RFC 8785 監査証跡)
  - `users.roles` / `users.organizationId` / `users.permissions` (RBAC 権限管理)
- **Firestore ルール実装規律**:
  上記 Server Only 領域の `allow write, create, update, delete` は Firestore ルール上 **`if false;`** とし、Cloud Functions / Admin SDK (サーバー権限) を経由しないいかなる変更も拒否する。

---

## ADR-018: Custom Claims Refresh Consistency (権限・テナント変更時のSSOT同期プロトコル)

```
Status:        Accepted (Frozen)
Date:          2026-08-01
Decision-Maker: Enterprise Architecture Team
Domain:        Security / RBAC
```

### 1. Context (背景と課題)
Firestore Security Rules で `request.auth.token.orgId` や `request.auth.token.roles` を参照することで高速 (O(1)) な判定が可能となるが、Firebase Auth の ID トークン (Custom Claims) は有効期限が最大1時間あり、DB 側の SSOT (`users/{uid}`) 変更後即座に反映されないタイムラグが生じる可能性がある。

### 2. Decision (決定事項)
権限変更（Role 変更、Tenant 変更、Billing 責任者変更）時における **4ステップ固定同期シーケンス (4-Step Claims Refresh Protocol)** を義務付ける。

```
[1. Firestore users/{uid} ドキュメントの SSOT 更新]
                        │
                        ▼
[2. RFC 8785 署名付き監査ログ (rbac-audit-logs) のアトミック生成]
                        │
                        ▼
[3. Firebase Admin SDK による setCustomUserClaims(uid, { orgId, roles }) 実行]
                        │
                        ▼
[4. クライアントに対する Token Refresh (idToken.getIdToken(true)) 要求]
```
これにより、SSOT ドキュメントと Auth Custom Claims 間の整合性を保証する。

---

## ADR-019: Financial Data Retention & Archive Policy (金融・監査データ保存とアーカイブ原則)

```
Status:        Accepted (Frozen)
Date:          2026-08-01
Decision-Maker: Enterprise Architecture Team & Security Board
Domain:        Governance / Compliance
```

### 1. Context (背景と課題)
Phase 6〜11.1 において、Invoice、Wallet Ledger、Accounting Journal、Audit Log 等の金融・監査データに対する不変性および完全性を実現した。しかし、地域スポーツクラブ・学校法人による本番 SaaS 運用を5年・10年と継続した場合、Firestore のストレージ容量増大・クエリインデックス性能の劣化、並びに過去監査証跡の保存コストおよび検索効率の低下が課題となる。

### 2. Decision (決定事項)
日本の法人税法および電子帳簿保存法要件、並びにシステム最適化要件を満たすため、以下の **3フェーズ財務・監査データ保存・アーカイブ原則 (3-Phase Financial Retention & Archive Policy)** を適用する。

```
[Active Period: 0〜24ヶ月]
 └─ 通常の Firestore アクティブコレクションで常時オンライン検索・対査可能
          │
          ▼
[Archive Period: 25ヶ月〜7年]
 └─ 読み取り専用の Cold Archive バケット (Cloud Storage / Read-Only Firestore) へ移行・保存
          │
          ▼
[Legal Retention Archive: 7年以上]
 └─ 法定保存義務期間経過後も、組織ポリシーに応じて長期署名検証アーカイブまたは正規エクスポート保持
```

- **不変ルール (Immutable Retention Rules)**:
  1. `❌ いかなるフェーズにおいても金融・監査データの物理削除 (Hard Delete) を禁止する。`
  2. `✅ Archive 移行時も RFC 8785 Canonical SHA-256 ハッシュチェーンを完全に維持する。`
  3. `✅ Read Only Archive および法定監査用エクスポート (CSV/JSONL) フォーマットへの無劣化変換を保証する。`

---

## ADR-020: External Integration Boundary Security (外部決済API非信頼ゾーン隔離原則)

```
Status:        Accepted (Frozen)
Date:          2026-08-01
Decision-Maker: Enterprise Architecture Team & Security Board
Domain:        Security / Payment
```

### 1. Context (背景と課題)
Stripe、GMOあおぞらネット銀行等の外部決済サービスから通知される Webhook イベントやレスポンスは、公衆インターネットを経由するためリプレイ攻撃やペイロード改ざんのリスクが存在する。外部決済事業者の型や状態仕様を直接 ASAHI ドメインモデルに反映させると、セキュリティホール及び仕様汚染を招く。

### 2. Decision (決定事項)
外部 API および Webhook 受信端点を **Untrusted Boundary (非信頼ゾーン)** として隔離する。
- `❌ 禁止`: Stripe/GMO 等のレスポンスによって直接 `Invoice` または `Wallet` を更新する行為 (`Stripe Response -> Invoice Update`)。
- `✅ 許可`: 必ず以下のセキュリティ分離パイプラインを経由してドメイン状態を変更すること。

```
External Webhook / API Response (Untrusted Zone)
        │
        ▼
[1. Webhook Signature Verifier] ─(HMAC 署名検証失敗時は即時切断・破棄)
        │
        ▼
[2. Provider Neutral Mapper] ───(Stripe/GMO型を PaymentIntentRecord へ中立正規化)
        │
        ▼
[3. PaymentCompletedEvent] ─────(ドメインイベント発行)
        │
        ▼
[4. Aggregate Mutation] ────────(Invoice.reconcileInvoice() による状態遷移)
```

---

## ADR-021: Payment Evidence Preservation (決済証跡・Webhook検証結果保存原則)

```
Status:        Accepted (Frozen)
Date:          2026-08-01
Decision-Maker: Enterprise Architecture Team & Security Board
Domain:        Audit / Compliance / Payment
```

### 1. Context (背景と課題)
会費請求等の決済が完了し `Invoice` が `PAID` に遷移した際、将来の監査・税務調査において「どの外部イベントおよび検証結果に基づき決済が成立したか」を数年後でも完全追跡・立証可能にする必要がある。

### 2. Decision (決定事項)
すべての決済トランザクションおよび Webhook 処理に対して、以下の属性を **不変決済監査証跡 (`PaymentEvidence`)** として記録・永続化する。
1. `providerEventId`: 外部サービスにおける一意イベントID（再送検出キー併用）
2. `providerTimestamp`: 外部サービスでの発生時刻
3. `signatureVerificationResult`: HMAC 署名検証ステータス (`VERIFIED` / `FAILED`)
4. `rawEventHash`: 受信ペイロード正規化文字列の SHA-256 ハッシュ
5. `processedAt`: ASAHI コアでの処理時刻
6. `adapterVersion`: 使用されたアダプタバージョン識別子

これにより、5年・10年後であっても、「この請求がなぜ PAID に遷移したか」を第三者監査人が技術的に検証・証明可能とする。



