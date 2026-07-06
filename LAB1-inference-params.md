# LAB1: 推論パラメータとPrompt Caching（60分・〜$1）

**狙い**: 問題集で頻出の temperature / top_p / maxTokens / stop sequences を「読んで知ってる」から「挙動を見た」に変える。後半で Prompt Caching のコスト効果を実測する。

対応ドメイン: D1（基盤モデル統合）・D4（コスト最適化）

## Part A: Playground でパラメータの挙動を見る（GUI）

Bedrock コンソール → Playgrounds → Chat/Text

1. モデル: Claude Haiku 系を選択
2. 同じプロンプトで以下を比較（各3回ずつ実行して**ばらつき**を見る）:

| 実験 | 設定 | 見るポイント |
|---|---|---|
| A-1 | temperature 0 vs 1 | 0はほぼ毎回同じ出力/1は毎回変わる |
| A-2 | temperature 1 + top_p 0.1 | temperatureが高くてもtop_pで絞ると安定する（両方で制御している） |
| A-3 | maxTokens 50 | **途中で切れる**（stopReason: max_tokens）。「出力が途中で切れる」トラブルの原因問題はこれ |
| A-4 | stop sequence に「。」 | 最初の文で止まる。エージェントのループ制御等での用途 |

プロンプト例: 「AWSの生成AIサービスを紹介する短いブログ記事を書いて」

3. 右側の**リクエスト/レスポンスJSON表示**を開き、実際のAPIパラメータ名を確認する
   （試験は `inferenceConfig` の中身を問うてくる）

## Part B: Converse API で再現（CLI）

```bash
# temperature 0（決定的）
aws bedrock-runtime converse --region us-east-1 \
  --model-id us.anthropic.claude-haiku-4-5-20251001-v1:0 \
  --messages '[{"role":"user","content":[{"text":"BedrockのGuardrailsを1文で説明して"}]}]' \
  --inference-config '{"temperature":0,"maxTokens":100}' \
  --query '{text: output.message.content[0].text, stop: stopReason, usage: usage}'
```

- 同じコマンドを2回打って出力が（ほぼ）同じことを確認
- `maxTokens: 30` にして `stopReason` が `max_tokens` に変わるのを確認
- `usage`（inputTokens/outputTokens）が**課金の実体**であることを確認

## Part C: Prompt Caching のコスト効果（CLI）

長いシステムプロンプト（>1024トークン程度の規約文書など）に `cachePoint` を置いて2回呼ぶ:

```bash
# 1回目: キャッシュ書き込み（cacheWriteInputTokens が付く）
# 2回目: キャッシュ読み込み（cacheReadInputTokens が付き、その分の単価が大幅割引）
```

Converse API の `system` に `{"cachePoint": {"type": "default"}}` を挟む。
レスポンスの `usage` で `cacheReadInputTokens` / `cacheWriteInputTokens` を見比べる。

## 確認ポイント（試験に出る粒度）

- [ ] temperature と top_p は**両方同時に**効く。決定的にしたいなら temperature=0
- [ ] 出力が途中で切れる → まず疑うのは maxTokens（stopReason で判別できる）
- [ ] Prompt Caching が効く条件: **プレフィックスが完全一致**・キャッシュポイントより前の部分・TTLは短い（数分）
- [ ] キャッシュが効くのは**入力側の単価**。出力トークンは割引されない
- [ ] InvokeModel と Converse の違い: Converse はモデル横断の統一フォーマット（試験は Converse 推し）

## 片付け

作るリソースなし。Playground/API呼び出しの従量課金のみ（Haikuなら合計$1未満）。
