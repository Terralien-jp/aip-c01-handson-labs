# LAB0: 環境準備＋コスト防衛線＋日本リージョン検証（45分・ほぼ¥0）

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

## 3. 日本リージョン検証（Claude を国内で使えるか）

実務（データレジデンシー要件）に直結する検証。**東京 (ap-northeast-1) でも**
Model access で Claude 系を有効化した上で、以下をCLIで確認する。

```bash
# 東京で提供されているClaudeモデルと推論タイプ
aws bedrock list-foundation-models --region ap-northeast-1 --by-provider anthropic \
  --query 'modelSummaries[].{id:modelId, inference:inferenceTypesSupported}' --output table

# 東京から使える推論プロファイル一覧（jp. で始まるものに注目）
aws bedrock list-inference-profiles --region ap-northeast-1 \
  --query 'inferenceProfileSummaries[?contains(inferenceProfileId, `anthropic`)].inferenceProfileId'

# jp.プロファイルで実呼び出し（東京発・処理は国内に留まる）
# ↑のlistに出たIDを使う（例: jp.anthropic.claude-haiku-4-5-...）
aws bedrock-runtime converse --region ap-northeast-1 \
  --model-id <jp.で始まるプロファイルID> \
  --messages '[{"role":"user","content":[{"text":"こんにちは。1文で自己紹介して"}]}]' \
  --inference-config '{"maxTokens":100}' \
  --query 'output.message.content[0].text'
```

### 確認ポイント（実務＋試験）

- [ ] 推論プロファイルのプレフィックスの意味:
  `us.`=米国内リージョン間 / `apac.`=アジア太平洋広域 / **`jp.`=東京↔大阪の日本国内のみ** / `global.`=全世界
- [ ] **`jp.` プロファイルは推論処理が日本国内（東京/大阪）に留まる**＝「データを国外に出せない」
  要件への公式解。apac. だと国外（シンガポール等）にルーティングされ得る点が決定的な違い
- [ ] 新しめのモデルは東京でのオンデマンド直呼び（モデルID指定）が無く、
  推論プロファイル経由のみの場合がある（`inferenceTypesSupported` で判別）
- [ ] クロスリージョン推論のクォータは**ソース（呼び出し元）リージョン単位**で管理される

## 4. 確認ポイント（試験に出る粒度）

- [ ] モデルアクセスは**リージョン単位**である（us-east-1で有効化しても東京では別）
- [ ] モデルIDと**推論プロファイル**（cross-region inference profile）の違い
- [ ] 課金の基本は**入出力トークン数**（モデルごと単価が違う。Haiku << Sonnet << Opus）

## 5. 片付け

このラボで作るのは予算アラートのみ（無料・残してよい）。日本リージョン検証の
converse 数回はHaikuなら1円未満。
