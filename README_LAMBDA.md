# Lambda 定時実行設定

この監視プログラムは CLI 実行に加えて、AWS Lambda から `monitor.lambda_handler` を呼び出せる。

## Lambda 関数の基本設定

- Runtime: Python 3.11 以上
- Handler: `monitor.lambda_handler`
- Timeout: 監視対象数に応じて 3〜15 分
- Memory: 256 MB 以上を推奨
- Environment variables:
  - `CONFIG_PATH`: 設定ファイルのパス。未指定時は `monitor.py` と同じディレクトリの `config.yaml`
  - `MONITOR_MODE`: `monitor` / `dry_run` / `self_test` / `probe_all`
  - `RAISE_ON_MONITOR_ERROR`: `true` にすると監視エラー時に Lambda 実行も失敗させる

## EventBridge Scheduler での定時実行

Lambda 側で時刻を決めて定時実行する場合は EventBridge Scheduler を使う。

例: 毎日 09:00 JST に実行する場合。

- Schedule pattern: `cron(0 9 * * ? *)`
- Timezone: `Asia/Tokyo`
- Target: この Lambda 関数
- Input:

```json
{
  "mode": "monitor"
}
```

事前疎通確認をしたい場合は Input を以下に変更する。

```json
{
  "mode": "self_test"
}
```

Slack へ通知せず結果だけ確認したい場合は以下を使う。

```json
{
  "mode": "dry_run"
}
```

## IAM 権限

S3 チェックを有効にする場合、Lambda 実行ロールに対象バケットの `s3:ListBucket` を付与する。

```json
{
  "Effect": "Allow",
  "Action": "s3:ListBucket",
  "Resource": "arn:aws:s3:::onsen-jasper-importer-prod-archive"
}
```

CloudWatch Logs への出力には、通常の Lambda 基本実行ロール `AWSLambdaBasicExecutionRole` が必要。

## VPC 設定

DB が VPC 内にある場合は、Lambda を DB に到達できる VPC / サブネット / セキュリティグループに配置する。

Slack Webhook へ送信するため、VPC 内 Lambda には NAT Gateway か外向き HTTPS 通信経路が必要。S3 へのアクセスは Gateway VPC Endpoint の利用も検討する。

## 依存ライブラリ

`requirements.txt` の依存を Lambda パッケージまたは Lambda Layer に含める。

`psycopg2-binary` はネイティブ依存を含むため、Lambda と互換性のある Amazon Linux 環境でビルドするか、コンテナイメージ Lambda の利用を推奨する。
