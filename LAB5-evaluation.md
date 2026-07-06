# LAB5: 評価とオブザーバビリティ（60分・〜$3）

**狙い**: 本番投入前に「品質をどう評価するか」「ハルシネーションをどう検知するか」「呼び出しをどう監査・監視するか」を実機で確認する。問題集の暗記知識（メトリクス名・設定項目名）を「コンソール/CLIで実際に触った」状態に変える。

対応ドメイン: D5（モデル評価とオブザーバビリティ・11%）中心、D4（コスト最適化・運用）関連分も一部カバー

## 前提

- LAB0 で Bedrock モデルアクセスを有効化済み（Anthropic Claude 系、Amazon Nova 系）
- リージョンは **us-east-1**、個人アカウント前提
- 評価データセットは**完全に架空の一般常識問題**（地理・歴史・科学）を使う。実データ・実案件は一切使わない

## Part A: Invocation logging を有効化して呼び出しを記録する

Bedrock の呼び出しログ（Model invocation logging）は**デフォルト無効**。有効化して初めて入出力とメタデータが記録される。

### A-1. CloudWatch Logs 宛のログ配信先を用意

```bash
# ロググループ作成
aws logs create-log-group --region us-east-1 \
  --log-group-name /bedrock/model-invocations-lab5

# IAMロール用の信頼ポリシー
cat > /tmp/bedrock-logging-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "bedrock.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "aws:SourceAccount": "YOUR_ACCOUNT_ID" },
        "ArnLike": { "aws:SourceArn": "arn:aws:bedrock:us-east-1:YOUR_ACCOUNT_ID:*" }
      }
    }
  ]
}
EOF

# ロールのアクセス権限（CreateLogStream/PutLogEvents）
cat > /tmp/bedrock-logging-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:YOUR_ACCOUNT_ID:log-group:/bedrock/model-invocations-lab5:log-stream:aws/bedrock/modelinvocations"
    }
  ]
}
EOF

aws iam create-role --role-name BedrockInvocationLoggingLab5Role \
  --assume-role-policy-document file:///tmp/bedrock-logging-trust.json

aws iam put-role-policy --role-name BedrockInvocationLoggingLab5Role \
  --policy-name BedrockLoggingPolicy \
  --policy-document file:///tmp/bedrock-logging-policy.json
```

`YOUR_ACCOUNT_ID` は `aws sts get-caller-identity --query Account --output text` で取得したものに置き換える。

### A-2. コンソールで invocation logging を有効化（GUI）

Bedrock コンソール → 左ペイン **Settings** → **Model invocation logging**

1. **Model invocation logging** をオン
2. ログに含めるモダリティ: **Text** にチェック（画像・埋め込み・動画は今回不要）
3. 配信先: **CloudWatch Logs only** を選択し、A-1 で作ったロググループとロールを指定
4. 保存

API/CLI での同等操作は `PutModelInvocationLoggingConfiguration`（今回は GUI 推奨。ロールARNの受け渡しが煩雑なため）。

### A-3. Playground で数回モデルを呼ぶ

Bedrock コンソール → Playgrounds → Chat/Text → Claude Haiku 系を選択し、以下のような一般知識の質問を3〜4回投げる。

例:
- 「富士山の標高は何メートルですか」
- 「フランスの首都はどこですか」
- 「光合成の仕組みを2文で説明してください」

### A-4. CloudWatch Logs で記録内容を確認

```bash
aws logs tail /bedrock/model-invocations-lab5 --region us-east-1 --since 10m
```

または CloudWatch Logs Insights で:

```bash
aws logs start-query --region us-east-1 \
  --log-group-name /bedrock/model-invocations-lab5 \
  --start-time $(date -v-30M +%s) --end-time $(date +%s) \
  --query-string 'fields timestamp, modelId, input.inputTokenCount, output.outputTokenCount | sort timestamp desc'
```

確認すること:
- ログエントリに `modelId`・`input.inputBodyJson`（プロンプト本文）・`output.outputBodyJson`（応答本文）・`input.inputTokenCount`・`output.outputTokenCount` が記録されている
- `identity.arn` に呼び出し元の IAM プリンシパルが自動で入る（自分で付与しなくてもよい）
- `requestMetadata` は呼び出し側が明示的に渡した場合のみ入る（今回のPlayground呼び出しでは空のはず）

## Part B: LLM-as-a-Judge で品質評価ジョブを実行する

### B-1. 評価用データセット（JSONL）を作成

架空の一般知識Q&Aを5問、`prompt` / `referenceResponse`（正解）/ `category` のキーで作る。これが Bedrock の model-as-a-judge 評価ジョブが要求する正式なフィールド名。

```bash
mkdir -p /tmp/lab5-eval
cat > /tmp/lab5-eval/dataset.jsonl <<'EOF'
{"prompt":"世界で一番面積が大きい国はどこですか。国名のみ答えてください。","category":"Geography","referenceResponse":"ロシア"}
{"prompt":"人体で最も大きい臓器は何ですか。","category":"Science","referenceResponse":"皮膚"}
{"prompt":"第二次世界大戦が終結した年は西暦何年ですか。","category":"History","referenceResponse":"1945年"}
{"prompt":"水の化学式を答えてください。","category":"Science","referenceResponse":"H2O"}
{"prompt":"日本で一番長い川の名前は何ですか。","category":"Geography","referenceResponse":"信濃川"}
EOF
```

（1データセットあたり最大1000プロンプトまで指定可能。今回は動作確認用に5問。)

### B-2. S3に配置

```bash
BUCKET=lab5-eval-$(aws sts get-caller-identity --query Account --output text)
aws s3 mb s3://$BUCKET --region us-east-1
aws s3 cp /tmp/lab5-eval/dataset.jsonl s3://$BUCKET/input/dataset.jsonl
```

### B-3. コンソールで評価ジョブを作成（GUI・推奨）

Bedrock コンソール → 左ペイン **Inference and assessment** → **Evaluations** → **Model evaluations** → **Create** → **Automatic: Model as a judge**

1. Evaluation name: 任意（例 `lab5-judge-eval`）
2. Evaluator model: **Amazon Nova 2 Lite**（判定役。生成モデルと別にする。LLM-as-a-judgeのビルトインメトリクスがサポートする判定モデルは `amazon.nova-2-lite-v1:0` 等の限定リストなので、コンソールの選択肢から選ぶこと）
3. Inference source: **Bedrock models** → 生成対象として **Claude Haiku 系** を選択
4. Metrics: 少なくとも1つ選択。今回は以下を選ぶ
   - **Correctness**（正確性。referenceResponseがあれば参照して判定）
   - **Completeness**（回答の網羅性）
   - **Faithfulness**（プロンプト/コンテキストにない情報を含んでいないか＝ハルシネーション検知の代理指標）
   - **Helpfulness**
5. Datasets: プロンプトデータセットに `s3://$BUCKET/input/dataset.jsonl` を指定、Evaluation results 出力先に別プレフィックス（例 `s3://$BUCKET/output/`）を指定
6. IAM role: **Create and use a new service role** を選択（評価ジョブに必要な権限をコンソールが自動付与）
7. Create でジョブ開始（数分〜十数分かかる）

### B-4. 同等操作をCLIで（参考）

```bash
cat > /tmp/lab5-eval/eval-job.json <<EOF
{
  "jobName": "lab5-judge-eval-cli",
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/YOUR_EVAL_SERVICE_ROLE",
  "applicationType": "ModelEvaluation",
  "evaluationConfig": {
    "automated": {
      "datasetMetricConfigs": [
        {
          "taskType": "General",
          "dataset": {
            "name": "lab5_dataset",
            "datasetLocation": { "s3Uri": "s3://$BUCKET/input/dataset.jsonl" }
          },
          "metricNames": ["Builtin.Correctness", "Builtin.Completeness", "Builtin.Faithfulness", "Builtin.Helpfulness"]
        }
      ],
      "evaluatorModelConfig": {
        "bedrockEvaluatorModels": [
          { "modelIdentifier": "amazon.nova-2-lite-v1:0" }
        ]
      }
    }
  },
  "inferenceConfig": {
    "models": [
      { "bedrockModel": { "modelIdentifier": "anthropic.claude-haiku-4-5-20251001-v1:0" } }
    ]
  },
  "outputDataConfig": { "s3Uri": "s3://$BUCKET/output/" }
}
EOF

aws bedrock create-evaluation-job --region us-east-1 --cli-input-json file:///tmp/lab5-eval/eval-job.json

# ジョブ状態確認
aws bedrock get-evaluation-job --region us-east-1 --job-identifier <JobArn>
```

**注**: 評価ジョブ用のIAMロール（`roleArn`）は、生成対象モデル・判定モデルの `InvokeModel` 権限とS3の読み書き権限を持つ必要がある。CLIで完結させたい場合は事前にロールを作る必要があり煩雑なため、初回は B-3 のコンソール手順（サービスロール自動作成）を推奨。

### B-5. 評価レポートを読む

ジョブが `Completed` になったら、コンソールの評価ジョブ詳細画面でメトリクスごとのスコア（レポートカード）を確認する。あわせて `s3://$BUCKET/output/` 配下に出力される詳細結果（各プロンプトごとのスコアと判定理由）も見る。

確認すること:
- 各メトリクスは内部的に5段階（または3〜7段階）のLikertスケールで判定され、0〜1に正規化されてレポートに出る
- Faithfulness が低いプロンプト＝プロンプトの文脈にない情報を回答に混ぜている＝ハルシネーションの代理指標として機能している
- Correctness / Completeness は `referenceResponse` を与えた場合、それを参照して判定される（与えなくても評価は可能）

## Part C: CloudWatchメトリクスでトークン数・レイテンシを見る

Bedrock ランタイムは `AWS/Bedrock` 名前空間に呼び出しごとのメトリクスを送っている。コスト監視・スロットリング検知の実務目線で見る。

### C-1. コンソールで確認

CloudWatch コンソール → Metrics → All metrics → **AWS/Bedrock** 名前空間 → `ModelId` ディメンションで Part A/B で呼んだモデルを選択

見るべき代表的メトリクス:

| メトリクス名 | 意味 |
|---|---|
| `Invocations` | 成功した呼び出し数（Converse/ConverseStream/InvokeModel系） |
| `InvocationLatency` | リクエスト送信〜最終トークン受信までの時間（ミリ秒） |
| `InvocationClientErrors` / `InvocationServerErrors` | クライアント側/サーバー側エラー数 |
| `InvocationThrottles` | スロットリングされた呼び出し数 |
| `InputTokenCount` / `OutputTokenCount` | 入力/出力トークン数（**課金の実体**） |
| `TimeToFirstToken` | ストリーミングAPIで最初のトークンが届くまでの時間 |

### C-2. CLIで取得

```bash
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/Bedrock \
  --metric-name InputTokenCount \
  --dimensions Name=ModelId,Value=anthropic.claude-haiku-4-5-20251001-v1:0 \
  --start-time $(date -v-1H -u +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum
```

`InputTokenCount` / `OutputTokenCount` を `Invocations` と突き合わせると「1回あたり平均トークン数」が出せる。コストの見積もり・急増検知（アラーム設定）の基礎になる。

## 確認ポイント（試験に出る粒度）

- [ ] **Model invocation logging はデフォルト無効**。有効化して初めて入出力・メタデータが記録される（AWS公式）
- [ ] Invocation logging の配信先は **CloudWatch Logs / S3 / 両方** から選べる。100KB超のボディやバイナリ（画像等）は S3 側にのみ格納される
- [ ] ログに記録される `identity.arn` は自動キャプチャ、`requestMetadata` は呼び出し側が任意で付与するタグである点は区別する
- [ ] Bedrock評価には大きく3種類ある: **Automatic（LLM含む自動評価）** / **Human-based（人手評価）** / その中の **Model as a judge（LLM-as-a-judge）**。RAGソース向けには別立てで **RAG evaluation**（Retrieve only / Retrieve and generate の2モード）がある
- [ ] LLM-as-a-judge は**生成モデル（generator）と判定モデル（evaluator）が別役割**。評価データセットは `prompt` / `referenceResponse`（任意）/ `category`（任意）のJSONL形式、S3配置、1ジョブ最大1000プロンプト
- [ ] ビルトインメトリクスの代表例: `Builtin.Correctness`（正確性）、`Builtin.Completeness`（網羅性）、`Builtin.Faithfulness`（プロンプト/文脈にない情報を含んでいないか＝**ハルシネーション検知の代理指標**）、`Builtin.Helpfulness`、`Builtin.Coherence`（論理的整合性）、`Builtin.Relevance`、`Harmfulness`/`Stereotyping`/`Refusal`（責任あるAI系メトリクス）
- [ ] `referenceResponse`（正解）を与えると Correctness / Completeness の判定に使われる。与えなくても評価は可能（無し版のプロンプトが別途用意されている）
- [ ] **評価ジョブ（LLM-as-a-judge・RAG評価）の判定モデル呼び出しは、オンデマンド標準ティア価格でトークン課金される**。アルゴリズム的スコアの算出自体には追加料金はないが、判定モデルの推論コストは別枠でかかる
- [ ] 人手評価（Human-based evaluation）を使うと、モデル推論コストに加えて**完了タスク1件あたり$0.21**が加算される
- [ ] CloudWatchの `AWS/Bedrock` 名前空間には `Invocations` / `InvocationLatency` / `InvocationClientErrors` / `InvocationServerErrors` / `InvocationThrottles` / `InputTokenCount` / `OutputTokenCount` / `TimeToFirstToken` 等がある。課金の実体は `InputTokenCount` / `OutputTokenCount`

## 片付け（このラボの後に必ず実行）

```bash
# 1. 評価ジョブの出力・入力データを削除してバケットごと削除
aws s3 rm s3://$BUCKET --recursive
aws s3 rb s3://$BUCKET

# 2. CloudWatch Logs のロググループ削除
aws logs delete-log-group --region us-east-1 \
  --log-group-name /bedrock/model-invocations-lab5

# 3. IAMロール・ポリシーの削除（A-1で作成した分）
aws iam delete-role-policy --role-name BedrockInvocationLoggingLab5Role --policy-name BedrockLoggingPolicy
aws iam delete-role --role-name BedrockInvocationLoggingLab5Role

# B-3でコンソールが自動作成した評価ジョブ用サービスロールも
# IAM コンソール → Roles で "AmazonBedrockExecutionRoleForModelEvaluation" 等の名前を検索して削除
```

最後に Bedrock コンソール → **Settings** → **Model invocation logging** を **OFF** に戻す（放置してもログ自体はCloudWatch Logs保持期間・S3ライフサイクル次第だが、ロググループ自体は上記で削除済みなので実害は小さい。習慣として毎回OFFに戻すこと）。

## 費用の目安（合計〜$3）

| 項目 | 目安 |
|---|---|
| Part A: Playground呼び出し（Claude Haiku、数回） | 数セント |
| Part B: 生成モデル（Claude Haiku）の推論（5プロンプト分） | 数セント |
| Part B: **判定モデル（Nova 2 Lite）の推論コスト**（5プロンプト×4メトリクス分の判定プロンプト） | 〜$1程度（メトリクス数×プロンプト数だけ判定呼び出しが走るため、素の生成コストより判定側の方がかさみやすい） |
| Part C: CloudWatchメトリクス取得・S3保存 | ごく僅か（無料枠内に収まることが多い） |
| 合計 | 〜$1〜3程度（判定モデルにSonnet系の高価なモデルを選ぶと増えるので、ラボでは意図的にNova 2 Liteを選定） |

判定モデルにSonnet/Opus系を選ぶと同じジョブでもコストが跳ねる点に注意。ラボの目的は「メトリクスの見え方」の確認なので、判定役は安価なモデル（Nova 2 Lite / Claude Haiku系）で十分。
