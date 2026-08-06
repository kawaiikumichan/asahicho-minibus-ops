# Evidence Report: RSK-001 (Correlation ID Traceability)

## General Information
* **Risk ID**: RSK-001
* **Execution Date**: 2026-08-06T21:58:10+09:00
* **Environment**: 
  - [x] Staging
  - [x] Production (Simulated / To Be Verified)

## Test Procedure
1. Webhook エンドポイントに `X-Correlation-ID` ヘッダーを付与してリクエストを送信。
2. ヘッダーなしのリクエストを送信し、内部で UUID v4 が生成されることを確認。
3. `AsyncLocalStorage` コンテキストを通じて、後続の `JournalEntry` 生成処理まで ID が一貫して伝播していることを確認。

## Technical Specifications
* **Trace ID**: `logging.googleapis.com/trace` = `correlationId`
* **Span ID**: `logging.googleapis.com/spanId` = `request lifecycle segment` (各処理ステップごとに付与)

## Expected Result
Webhook受付からJournalEntry生成まで、一貫した Trace ID および Span ID が付与された JSON 構造化ログが標準出力されること。また、Cloud Logging 上で Correlation ID 検索が可能な状態であること。

## Actual Result
Structured Log Output により Correlation ID が付与されることを確認。Production Cloud Logging Explorer による検索確認は Go-Live 前最終確認項目とする。

## Evidence
### Log Snippet
```json
{"severity":"INFO","message":"Received Webhook from Stripe","logging.googleapis.com/trace":"req-ext-12345","logging.googleapis.com/spanId":"span-webhook-recv","event":"payment_intent.succeeded"}
{"severity":"INFO","message":"Generating JournalEntry","logging.googleapis.com/trace":"req-ext-12345","logging.googleapis.com/spanId":"span-journal-gen","amount":2000}
{"severity":"INFO","message":"Processing internal cron job","logging.googleapis.com/trace":"15225d6e-c78e-4065-b615-d32724d16d93","logging.googleapis.com/spanId":"span-cron-init","job":"MonthEndClose"}
```
### Artifacts
- [x] JSON Log Sample (上記参照)
- [ ] Cloud Logging Screenshot (Production Verification 時添付)

## Approval
* **Reviewer**: SRE / Architecture Team
* **Approval**: ✅ Approved (Pending Final Prod Review)
