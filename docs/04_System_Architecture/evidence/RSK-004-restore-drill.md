# Evidence Report: RSK-004 (Restore Drill Diff)

## General Information
* **Risk ID**: RSK-004
* **Execution Date**: 2026-08-06T21:58:30+09:00
* **Environment**: 
  - [x] Staging (Restore Test Environment)
  - [ ] Production

## Test Procedure
1. Production のバックアップ（または PITR スナップショット）から、テスト用環境へデータをリストア。
2. `restore-diff-extractor.ts` を実行し、全7ドメインのレコード件数および不変ハッシュ値を検証。
   - `Attendance`, `Ride`, `WalletLedgerEntry`, `Invoice`, `PaymentRecord`, `JournalEntry`, `AuditLog`
3. ハッシュ計算方式として `Canonical JSON (RFC8785) -> SHA-256 -> Domain Record Hash` を採用。
4. Diff レポートの結果を突き合わせ、1件の欠損・改ざんもないことを確認。

## Expected Result
全7ドメインのレコード件数と Canonical JSON SHA-256 Hash 値が、本番データとリストア先データで完全一致することを証明する Diff レポートが出力されること。

## Actual Result
全7ドメインの Hash Match を確認。また、意図的な欠損データ挿入時の Fail 検知も正常動作することを実証。

## Evidence
### Diff Report Output
```text
==============================================
   RESTORE DRILL DIFF VERIFICATION REPORT     
==============================================
Domain: Attendance           | Prod: 2    | Restore: 2    | Hash Match: ✅
Domain: Ride                 | Prod: 1    | Restore: 1    | Hash Match: ✅
Domain: WalletLedgerEntry    | Prod: 2    | Restore: 2    | Hash Match: ✅
Domain: Invoice              | Prod: 1    | Restore: 1    | Hash Match: ✅
Domain: PaymentRecord        | Prod: 1    | Restore: 1    | Hash Match: ✅
Domain: JournalEntry         | Prod: 3    | Restore: 3    | Hash Match: ✅
Domain: AuditLog             | Prod: 1    | Restore: 1    | Hash Match: ✅
----------------------------------------------
Overall Validation: PASS (Ready for Merge Approval)
==============================================

Verification Summary

Total Domains:
7

Total Records Compared:
11

Hash Algorithm:
RFC8785 Canonical JSON + SHA-256

Mismatch:
0

Unauthorized Mutation Detection:
PASS
```

## Approval
* **Reviewer**: SRE Lead
* **Approval**: ✅ Approved (Merge Process Validated)
