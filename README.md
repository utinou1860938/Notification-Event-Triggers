# Notification-Event-Triggers

AWS SES（Email）および AWS End User Messaging（SMS）の通知イベントを SNS → SQS で受け取れることを検証する POC リポジトリ。

## 目的

SES / EUM が発行するイベント（バウンス、配信失敗など）を本当にキャッチできるのかを実際に動かして確認する。

## アーキテクチャ

### POC（本リポジトリのスコープ）

```
SES / EUM → SNS → SQS（コンシューマーなし）
```

SQS にメッセージが入ってきたことを確認し、メッセージの中身を確認できたらゴール。

### プロダクション（最終形イメージ）

```
SES / EUM → SNS → SQS → ECS（ロングポーリング） → DB 更新
```

## 対象イベント

### SES（Email）

| イベント | 意味 |
|---------|------|
| Bounce | 宛先不明・メールボックス満杯など |
| Complaint | 受信者がスパム報告 |
| Delivery | 正常に配信された |

### EUM（SMS）

| イベント | 意味 |
|---------|------|
| TEXT_SUCCESSFUL | 正常に配信された |
| TEXT_FAILED | 送信失敗（汎用） |
| TEXT_TTL_EXPIRED | 配信タイムアウト |
| TEXT_BLOCKED | メッセージがブロックされた |
| TEXT_CARRIER_BLOCKED | キャリア側で拒否 |
| TEXT_INVALID | 無効な電話番号 |
| TEXT_CARRIER_UNREACHABLE | キャリアに到達できない |
| TEXT_UNKNOWN | 不明なエラー |

## イベント再現方法

### SES

| イベント | 再現方法 |
|---------|---------|
| Bounce | SES メールボックスシミュレーター（`bounce@simulator.amazonses.com`） |
| Complaint | SES メールボックスシミュレーター（`complaint@simulator.amazonses.com`） |
| Delivery | 個人メールアドレス宛に送信 |

### EUM（SMS）

| イベント | 再現方法 |
|---------|---------|
| Delivery | 個人番号宛に送信 |
| Blocked 系 | 個人番号で受信後「STOP」返信 → 再送信（要検証） |
| その他 Failure 系 | 要検証（シミュレーター機能なし） |

> **注意**: SMS はシミュレーター機能がないため、テスト送信でも実際の料金が発生する。

## インフラ

CloudFormation で構築する。
ドメインなどの手動作成のリソースは外部パラメータとして受け取る。

### 認証確認

```bash
aws sts get-caller-identity
```

### 変更セット作成（ドライラン）

```bash
aws cloudformation deploy \
  --template-file template.yml \
  --stack-name notification-event-triggers \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
  DomainName=<ドメイン名> \
  HostedZoneId=<ホストゾーンID> \
  --no-execute-changeset
```

### デプロイ

```bash
aws cloudformation deploy \
  --template-file template.yml \
  --stack-name notification-event-triggers \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
  DomainName=<ドメイン名> \
  HostedZoneId=<ホストゾーンID> \
```

### 削除

```bash
aws cloudformation delete-stack \
  --stack-name notification-event-triggers
```

### SES 送信制御ポリシー（任意：手動作成）

SES コンソール → Identity（ドメイン） → Authorization タブ → Create Policy で以下を設定する。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<アカウントID>:root"
      },
      "Action": [
        "ses:SendEmail",
        "ses:SendRawEmail"
      ],
      "Resource": "arn:aws:ses:<リージョン>:<アカウントID>:identity/<ドメイン名>",
      "Condition": {
        "StringLike": {
          "ses:FromAddress": "*@<ドメイン名>"
        },
        "ForAllValues:StringLike": {
          "ses:Recipients": [
            "<許可する送信先1>",
            "<許可する送信先2>",
            "<許可する送信先3>",
            "*@simulator.amazonses.com"
          ]
        }
      }
    }
  ]
}
```

### 検証（SNSへメッセージを送信）

```bash
aws sns publish \
    --topic-arn <トピックARN> \
    --message '{"test": "hello"}'
```

### 参考にした記事
- [Amazon SESで送信元と宛先の制限をかけてみたメモ](https://qiita.com/kumeneko/items/423bcf2d0fdefbd54334)
- [SPFとは？SPFの意味やDKIM・DMARCとの違いを分かりやすく解説](https://am.arara.com/blog/06)
- [IPレピュテーション・ドメインレピュテーションとは？ 違いや確認方法、向上のポイントを解説](EiwAkZgxixSb59tDajc7hM8U45d8CbxhGOJO9JykHJjKMIFp8JJWrxVWTIs0pxoCvxQQAvD_BwE)
- [Amazon SES の新機能 Virtual Deliverability Manager を使ってみた](https://dev.classmethod.jp/articles/amazon-ses-vdm/)

## ゴール

- [ ] SES イベント（Bounce / Complaint / Delivery）が SQS に届くことを確認
- [ ] EUM イベント（Delivery / Failure 系）が SQS に届くことを確認
- [ ] 各メッセージの JSON 構造を確認・記録
