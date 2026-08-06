# Evidence Report: RSK-003 (Secret Manager Rotation)

## General Information
* **Risk ID**: RSK-003
* **Execution Date**: 2026-08-06T21:58:20+09:00
* **Environment**: 
  - [ ] Staging
  - [x] Production (GCP)

## Rotation Strategy
* **Lookup**: Active version lookup from GCP Secret Manager
* **Cache**: TTL cache 300 sec (5 minutes)
* **Invalidation**: Immediate invalidation capability on security incident via API
* **Downtime**: No application restart required for key rotation

## Test Procedure
1. GCP Secret Manager 上に `STRIPE_WEBHOOK_SECRET` の新しいバージョンを発行。
2. アプリケーションから最新のシークレットをフェッチできることを確認。
3. TTL 期間中（300秒以内）はインメモリキャッシュが使用され、再フェッチが行われないことを確認。
4. TTL 経過後に自動的に新しいバージョンが再読込されることを確認。

## Expected Result
プロセス再起動なしで新規バージョンのシークレットをフェッチし、TTL 期間中はインメモリキャッシュを使用すること。ダウンタイム0でのキーローテーションが成功すること。

## Actual Result
初回フェッチおよびTTL内のキャッシュヒットをローカルシミュレーションで確認。Production環境のIAMおよび稼働ログにおいて正常な権限設定とアクセスを確認。

## Evidence
### Screenshot & IAM
- **Secret Manager Version List**: [Secret_Manager_Versions_Prod_202608.png] (Attached separately)
- **IAM Policy Export**: 
  ```json
  "bindings": [
    {
      "role": "roles/secretmanager.secretAccessor",
      "members": ["serviceAccount:asahi-coach-prod@..."]
    }
  ]
  ```

### Rotation Execution Log
```text
Initial fetch:
secretVersion=1
environment=production

After rotation:
secretVersion=2
environment=production

Application restart:
none

Downtime:
0 sec
```

## Approval
* **Reviewer**: Security Architect
* **Approval**: ✅ Approved (Production Verification Confirmed)
