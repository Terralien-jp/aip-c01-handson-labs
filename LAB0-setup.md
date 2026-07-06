# LAB0: 環境準備＋コスト防衛線（30分・¥0）

全ラボの前提。**コスト防衛線を先に張ってから**Bedrockに触る。

## 1. 予算アラート（必須・最初にやる）

GUI: Billing and Cost Management → Budgets → Create budget

- テンプレート「月次コスト予算」でOK
- 金額: **$20**（ラボ全体の上限想定）
- アラート: 実績50% / 実績80% / **予測100%** の3段でメール通知

CLI版（2回目以降の再現用）:
```bash
aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{"BudgetName":"handson-guard","BudgetLimit":{"Amount":"20","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"あなたのメール"}]}]'
```

## 2. Bedrock モデルアクセス有効化

GUI: Bedrock コンソール（us-east-1）→ Model access → Enable specific models

ラボで使う最小セット:
- **Anthropic Claude（Haiku系）** … メインの推論用（安い・速い）
- **Amazon Titan Text Embeddings V2** … LAB2 の埋め込み用
- **Amazon Nova 2 Lite**（`amazon.nova-2-lite-v1:0`） … 比較用の第2モデル（LAB5のLLM-as-a-Judge判定モデルにも使う）

Anthropicモデルは初回にユースケース記入を求められる場合がある（個人学習でOK）。
承認は通常数分。

## 3. 確認ポイント（試験に出る粒度）

- [ ] モデルアクセスは**リージョン単位**である（us-east-1で有効化しても東京では別）
- [ ] モデルIDと**推論プロファイル**（cross-region inference profile, `us.` プレフィックス）の違い
  - 新しめのモデルはオンデマンドが推論プロファイル経由のみの場合がある
- [ ] 課金の基本は**入出力トークン数**（モデルごと単価が違う。Haiku << Sonnet << Opus）

## 4. 片付け

このラボで作るのは予算アラートのみ（無料・残してよい）。
