# AIP-C01 ハンズオンラボ集（非公式）

AWS Certified Generative AI Developer – Professional（AIP-C01）対策の**実機ハンズオン教材**。
問題集や座学で「解ける」けど「触ったことない」を潰すことに特化し、試験ドメインの中でも
**設定値の挙動レベルで問われる深いゾーン**（チャンク戦略・Guardrails設定・推論パラメータ・評価）に絞って構成しています。

- 2026年7月時点のAWS公式ドキュメントで裏取り済み（LAB4のBedrock Agents Classicの扱いは2026年8月に再確認・更新）
- 全ラボに**費用目安と片付け手順**付き（合計6〜7時間・$10未満で完走可能）
- 手順は「GUIで概念を掴む → CLI/コードで再現」の二段構成

> **免責**: 本教材は非公式です。AWSの画面・仕様・料金は変わります。発生する利用料金は自己負担です。
> 必ずLAB0の予算アラートを設定してから始めてください。

## ラボ一覧

| # | ラボ | 試験ドメイン | 時間 | 費用目安 |
|---|---|---|---|---|
| 0 | [環境準備＋コスト防衛線＋日本リージョン検証](LAB0-setup.md) | 前提 | 45分 | ほぼ¥0 |
| 1 | [推論パラメータとPrompt Caching](LAB1-inference-params.md) | D1/D4 | 60分 | 〜$1 |
| 2 | [Knowledge Base とチャンク戦略](LAB2-knowledge-base.md) | D1(31%) | 90分 | 〜$3 ⚠️片付け厳守 |
| 3 | [Guardrails 全機能](LAB3-guardrails.md) | D3(20%) | 60分 | 〜$1 |
| 4 | [Agents / Action Group](LAB4-agents.md) ⚠️**アカウントにより実行可否が分かれる（下記）** | D2(26%) | 90分 | 〜$1 |
| 5 | [評価とオブザーバビリティ](LAB5-evaluation.md) | D5(11%)+D4 | 60分 | 〜$3 |

## ⚠️ LAB4 の前提が変わりました（2026-07-30 以降）

**Amazon Bedrock Agents（Classic）は 2026年7月30日にメンテナンスモードへ入り、新規顧客への提供を終了しました。**
以降、**過去12か月に Bedrock Agents の利用実績がないアカウント**では `CreateAgent` と `InvokeInlineAgent` が
`AccessDeniedException`（HTTP 403）になります。**例外申請の窓口はありません**（AWSが利用実績で自動判定します）。

- **利用実績のあるアカウント**: 影響なし。LAB4 をそのまま実施できます
- **実績のないアカウント**: LAB4 の Classic 手順は実行できません。**[LAB4 付録の AgentCore 版](LAB4-agents.md#付録-agentcore-で同じことをやる新規アカウント向け)** に進んでください
- 既存エージェントの運用（`UpdateAgent` / `InvokeAgent` / `PrepareAgent` / 各種 Get・List・Delete）は**全アカウントで引き続き利用可能**。移行期限も end-of-life の予定日も公表されていません
- ただし **Classic のモデルカタログは 2026年7月30日で凍結**され、以降の新モデルは AgentCore 側にのみ来ます。新規開発の推奨先は **Amazon Bedrock AgentCore** です

出典: [Amazon Bedrock Agents Classic maintenance mode](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html)（AWS公式）

## ⚠️ コスト地雷トップ3（先に知っておく）

1. **Knowledge Base の quick create → OpenSearch Serverless**: 最小構成でも放置すると**月3〜5万円級**。
   LAB2 では S3 Vectors を使います（桁違いに安い）。OpenSearch を作った場合は当日中に削除
2. **Provisioned Throughput**: 購入した瞬間から時間課金。**ラボでは一切購入しません**（試験知識としては「知っているだけ」で足ります）
3. **Fine-tuning ジョブ**: 1回数千円〜。試験ウェイトに対して割に合わないため本ラボ集では扱いません

## 進め方のおすすめ

順番どおりでなくてOKです。**LAB0（予算アラート）だけ必ず最初**に。
あとは模試や問題集で間違えた分野のラボからやる「間違い駆動」が効率的です。

## 読む教材: AIP-C01 サービス別ノート

このラボ集が「触って覚える」側だとすると、読む側はこちらにあります。

**[AIP-C01 サービス別ノート](https://terralien.net/learn/aip-c01/)** — 出題範囲の113サービスを1つずつ、
「要するに何か・なぜ生まれたか・何に使うか・試験でどう問われるか・実務でどう使うか・喩え・想起チェック」で整理したもの。
出典は **AWS 公式ドキュメントのみ**（機械チェックで非公式ドメインを弾いています）。

**ノートの誤り・古くなった記述は、このリポジトリの Issue で受け付けています。**
各ノートの末尾に報告ボタンがあり、対象ページが埋まった状態で Issue を作れます。
AWS の仕様は変わるので、**「当時は正しかったが今は違う」という指摘も歓迎**です。

## ライセンス

MIT（[LICENSE](LICENSE)）。ラボ・ノートいずれについても、誤りの指摘・仕様変更の報告は Issue へどうぞ。
