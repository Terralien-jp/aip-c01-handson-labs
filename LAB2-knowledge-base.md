# LAB2: Knowledge Base とチャンク戦略（90分・〜$3 ⚠️片付け厳守）

**狙い**: 問題集で頻出の「チャンク戦略の選定（fixed / semantic / hierarchical のどれを選ぶか）」「検索精度が悪い時の対処」を、実際に同じデータ・同じ質問で3種類のチャンク戦略を比較して体で覚える。あわせて Retrieve API と RetrieveAndGenerate API の違い、データソース同期の挙動を実機で確認する。

対応ドメイン: D1（基盤モデル統合, 31%）

## ⚠️ コスト地雷の警告（最初に読む）

Bedrock Knowledge Base のベクトルストアには **Amazon S3 Vectors** を使う。2026年7月時点で S3 Vectors は GA（一般提供）済みで、Bedrock Knowledge Base 作成画面から直接選択できる（[S3 Vectors integration, docs.aws.amazon.com](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html)）。ストレージ課金は約 $0.06/GB/月＋リクエスト従量制で、**「作りっぱなしでも桁が変わるほど怖くない」**。

これに対して、KB作成画面でうっかり **OpenSearch Serverless** を選ぶと話が変わる。OpenSearch Serverless はコレクションが起動している間、OCU（OpenSearch Compute Unit）単位で**常時課金**される。最小構成でも実質 **1時間あたり数十円〜、放置すると月3〜5万円規模**になる。このラボでは選ばない。

- 本ラボは **必ず「S3 vector bucket」をベクトルストアとして選ぶこと**（作成画面で明示的に選択肢がある）
- 誤って OpenSearch Serverless ベースの Knowledge Base を作ってしまった場合は、**その日のうちにコレクションごと削除**する（片付けセクション参照）
- 片付けセクションは本ラボで**最重要**。90分の作業より片付けの5分を優先する

## 事前準備

- LAB0 で Titan Text Embeddings V2 のモデルアクセスが有効化済みであること
- テスト用テキストファイルを3〜5個、手元で自作する（実データ・社内文書は絶対に使わない。以下のようなダミーで十分）

```bash
mkdir -p ~/kb-lab/docs
cat > ~/kb-lab/docs/doc1-onboarding.txt << 'EOF'
【架空社内規程】新入社員オンボーディング手順
第1章 初日の流れ
新入社員は初日に人事部でIDカードを受け取る。次にIT部門でノートPCとアカウントの発行を受ける。
午後は配属先チームとのオリエンテーションを行う。
第2章 1週間目
1週間目は各種研修を受講する。研修にはセキュリティ研修、コンプライアンス研修、社内システム研修が含まれる。
研修終了後、簡単な理解度確認テストを実施する。
第3章 1ヶ月目の評価
1ヶ月経過時点でマネージャーと1on1を実施し、業務理解度と適応状況を確認する。
EOF

cat > ~/kb-lab/docs/doc2-expense.txt << 'EOF'
【架空社内規程】経費精算ルール
交通費は実費精算とし、領収書の添付を必須とする。
出張時の宿泊費は1泊あたり上限15000円とする。上限を超える場合は事前申請が必要。
飲食を伴う会議費は1人あたり上限3000円とし、参加者名を申請書に記載すること。
経費精算は発生から30日以内に申請しなければならない。30日を超えた申請は原則却下される。
EOF

cat > ~/kb-lab/docs/doc3-remote.txt << 'EOF'
【架空社内規程】リモートワーク規程
週3日までのリモートワークを許可する。事前にマネージャーの承認を得ること。
リモートワーク日は勤怠システムに事前登録し、コアタイム（10時〜15時）は常時連絡が取れる状態を維持する。
セキュリティ上、社外のフリーWi-Fiでの業務利用は禁止する。VPN接続を必須とする。
EOF

cat > ~/kb-lab/docs/doc4-vacation.txt << 'EOF'
【架空社内規程】休暇取得ルール
年次有給休暇は入社半年後に10日付与される。以後1年ごとに勤続年数に応じて加算される。
連続5日以上の休暇取得を希望する場合は、取得予定日の2週間前までに申請する。
繁忙期（決算期の月末最終週）は原則として休暇取得を控えるよう推奨される。
EOF
```

（ドメイン名・社名・部署名などは全て仮名。中身の分量は数百トークン程度で十分＝チャンク戦略の違いが見えるサイズ感）

## Part A: GUIでKB作成（チャンク戦略はまず fixed-size）

1. S3コンソールでバケットを新規作成（例: `kb-lab-<自分のアカウントID>-src`）し、`~/kb-lab/docs/` の4ファイルをアップロード
2. Bedrock コンソール（us-east-1）→ **Knowledge Bases** → **Create** → **Knowledge Base with vector store**
3. Knowledge Base name は任意（例: `kb-lab-fixed`）。IAM permissions は「新しいサービスロールを作成」のデフォルトのままでよい
4. データソースタイプ: **Amazon S3** を選択 → Next
5. **Configure data source** 画面
   - S3 URI: 手順1で作ったバケットを指定
   - Parsing strategy: デフォルト（Amazon Bedrock default parser）のまま
   - **Chunking strategy**: **Fixed-size chunking** を選択し、以下を設定
     - Max tokens per chunk: `300`（デフォルト値でよい）
     - Overlap percentage: `20`
6. **Embeddings model と Vector store** 画面
   - Embeddings model: **Titan Text Embeddings V2** を選択（Vector dimensions はデフォルトの1024でよい。S3 Vectors統合には floating-point 形式が必須で、Titan V2 はこれに対応）
   - **Vector store**: **Quick create a new vector store** を選び、ストアの種類として **S3 vector bucket** が選択されていることを確認（OpenSearch Serverless になっていないか必ず目視確認）
7. 内容を確認して **Create Knowledge Base**
8. 作成完了後、Knowledge Base の詳細画面 → データソースを選択 → **Sync** を押してインジェスト（同期）を実行
   - 裏側では `StartIngestionJob` が発行される。ステータスが `STARTING` → `IN_PROGRESS` → `COMPLETE` と遷移するのをコンソールで確認（数十秒〜数分）

同期完了後、コンソール右側の **Test Knowledge Base** ペインで質問してみる。

- 質問例: 「経費精算の申請期限は？」「リモートワークは週何日まで？」
- 返ってきた回答の下に表示される **Source chunks** を開き、実際に検索でヒットしたチャンクのテキスト範囲を確認する

## Part B: チャンク戦略を変えて比較（semantic / hierarchical）

同じ4ファイルに対して、チャンク戦略だけを変えた Knowledge Base をもう2つ作る（データソースは同じS3バケットを使い回してよい。KBを新規に作り直す）。

### KB-2: Semantic chunking

Part A の手順4〜6と同様に進め、Chunking strategy で **Semantic chunking** を選択。設定項目:

- Max tokens: `300`
- Buffer size: `1`（対象の文の前後1文ずつを合わせて埋め込みを作る＝前後合わせて3文で意味的な区切りを判定する）
- Breakpoint percentile threshold: `95`（デフォルト値。値が高いほど「よほど意味が変わらない限り区切らない」＝チャンクが大きく・少なくなる）

Vector store は同じく **S3 vector bucket** を Quick create。

### KB-3: Hierarchical chunking

**注意**: 公式ドキュメントに明記されている通り、[hierarchical chunking は S3 vector bucket をベクトルストアに使う場合は非推奨](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)（親子チャンクの階層情報がフィルタ不可メタデータとして保存されるため、トークン数が大きいと1ベクトルあたりのメタデータサイズ上限を超える可能性がある）。このラボのような小さいサンプルデータでは実害が出にくいので試すこと自体は可能だが、**本番で hierarchical を使うなら OpenSearch Serverless 等の別ベクトルストアを検討する**、という試験知識として押さえておく。

Chunking strategy で **Hierarchical chunking** を選択。設定項目:

- Parent max tokens: `1500`
- Child max tokens: `300`
- Overlap tokens: `60`

### 比較する

3つのKBそれぞれで **Test Knowledge Base** から同じ質問を投げ、返ってくるチャンクの粒度を見比べる。

- 質問例: 「新入社員研修にはどんな種類がありますか、また評価はいつ行われますか」（doc1の複数章にまたがる内容）
- Fixed-size: 300トークンで機械的に切られているため、「研修の種類」と「1ヶ月評価」が別チャンクに分断されていないか確認
- Semantic: 意味的なまとまりで区切られるため、章の境界に近い場所で切れているか確認
- Hierarchical: 子チャンク（300トークン）でヒットしても、実際にモデルに渡るのは親チャンク（1500トークン）に差し替えられている点を確認（レスポンスの取得件数が指定数より少なく見えることがある＝子が同じ親に統合されるため）

## Part C: CLIで Retrieve / RetrieveAndGenerate を叩く

Knowledge Base ID はコンソールの詳細画面、または以下で取得できる。

```bash
aws bedrock-agent list-knowledge-bases --region us-east-1 \
  --query 'knowledgeBaseSummaries[].{id:knowledgeBaseId,name:name}'
```

### Retrieve API（検索結果のみ。生成なし）

```bash
aws bedrock-agent-runtime retrieve --region us-east-1 \
  --knowledge-base-id <KB_ID> \
  --retrieval-query '{"text":"経費精算の申請期限は？"}' \
  --retrieval-configuration '{"vectorSearchConfiguration":{"numberOfResults":5}}' \
  --query 'retrievalResults[].{score:score,text:content.text}'
```

- レスポンスの `retrievalResults` 配列の各要素に `score`（関連度スコア、0〜1のfloat）と `content.text`（実際に取得されたチャンクの中身）が入る
- `location.s3Location.uri` でどのファイルから取れたかが分かる
- 3つのKB（fixed/semantic/hierarchical）で同じクエリを打ち、`score` の分布とチャンクの切れ方を見比べる

### RetrieveAndGenerate API（検索＋生成を1コールで）

```bash
aws bedrock-agent-runtime retrieve-and-generate --region us-east-1 \
  --input '{"text":"経費精算の申請期限と、上限を超えた場合の扱いを教えて"}' \
  --retrieve-and-generate-configuration '{
    "type": "KNOWLEDGE_BASE",
    "knowledgeBaseConfiguration": {
      "knowledgeBaseId": "<KB_ID>",
      "modelArn": "us.anthropic.claude-haiku-4-5-20251001-v1:0",
      "retrievalConfiguration": {
        "vectorSearchConfiguration": {"numberOfResults": 5}
      }
    }
  }' \
  --query '{answer: output.text, citations: citations}'
```

- `output.text` に生成された回答本文が入る
- `citations` 配列の各要素に、回答のどの部分がどのチャンク（`retrievedReferences`）を根拠にしているかの対応関係が入る。**根拠なき部分は引用されない**＝ハルシネーション抑制の仕組みを実データで確認する
- Retrieve は「検索結果を自分のアプリで料理したい場合」、RetrieveAndGenerate は「検索から回答生成まで丸ごとBedrockに任せたい場合」に使う、という使い分けを実際のレスポンス構造の違いで体感する

### データソース再同期の確認（任意・時間があれば）

`~/kb-lab/docs/doc2-expense.txt` の内容を書き換えて再度アップロードし、コンソールから **Sync** を実行するか、CLIで叩く。

```bash
aws bedrock-agent start-ingestion-job --region us-east-1 \
  --knowledge-base-id <KB_ID> --data-source-id <DS_ID>

# ジョブIDを使ってポーリング
aws bedrock-agent get-ingestion-job --region us-east-1 \
  --knowledge-base-id <KB_ID> --data-source-id <DS_ID> --ingestion-job-id <JOB_ID> \
  --query '{status:ingestionJob.status, stats:ingestionJob.statistics}'
```

- `StartIngestionJob` は**非同期**。呼んだ直後は `STARTING`、その後 `IN_PROGRESS` → `COMPLETE`（失敗時は `FAILED`）と遷移する
- `statistics` に `numberOfDocumentsScanned` / `numberOfModifiedDocumentsIndexed` / `numberOfNewDocumentsIndexed` / `numberOfDocumentsFailed` 等が入り、**差分だけが再インデックスされたこと**（変更していない doc1/doc3/doc4 はスキャンはされるが再エンベディングはされない）が確認できる

## 確認ポイント（試験に出る粒度）

- [ ] チャンク戦略は **fixed-size / semantic / hierarchical / no chunking（default含む）** の4系統。Default chunkingは約300トークン・文境界を尊重した固定サイズの一種
- [ ] Fixed-size は「トークン数」と「overlap percentage」を指定。仕組みが単純な分、**意味的なまとまりが機械的に分断されるリスク**がある（本ラボPart Bで実見した通り）
- [ ] Hierarchical が効くのは**構造化された長文書**（マニュアル・法務文書・複雑な表を含む学術論文など）。子チャンクで検索の精度を保ちつつ、親チャンクで文脈の広さを確保する二段構え
- [ ] **Hierarchical chunking は S3 Vectors では非推奨**（親子階層がフィルタ不可メタデータとして持たれ、トークン数が大きいとメタデータサイズ上限に抵触しうる）。本番で使うなら別のベクトルストアを検討する
- [ ] Semantic chunking は埋め込み生成に加えて**追加の基盤モデル呼び出しコストがかかる**（文の意味的な区切りを判定するため）
- [ ] チャンクが大きすぎると精度が落ちる理由: 1チャンクに複数トピックが混在し、埋め込みベクトルが「どっちつかず」になって類似度検索がぼやける。小さすぎると今度は文脈が失われる（trade-off）
- [ ] **Knowledge Base の埋め込みモデルは作成後に変更できない。** `UpdateKnowledgeBase` で変更できるのは name / description / roleArn のみで、埋め込みモデルを含む `knowledgeBaseConfiguration` は作成時と同じ値を再送する必要がある（＝事実上変更不可）。**「埋め込みモデルを変えたら再同期すればよい」は誤り。正しくはKBを作り直す（再作成）**しかない。これは試験のひっかけとして狙われやすい
- [ ] データソース同期（`StartIngestionJob`）は非同期ジョブ。ステータスは `STARTING/IN_PROGRESS/COMPLETE/FAILED/STOPPING/STOPPED`。**差分のみ再インデックス**される（変更ファイルだけ再エンベディング）
- [ ] Retrieve API は検索結果（チャンク＋スコア）のみを返す。RetrieveAndGenerate API は検索＋LLM生成＋citationsまで1コールで返す
- [ ] RetrieveAndGenerate のレスポンスの `citations` は、回答文のどの区間がどの取得チャンクを根拠にしているかを対応付ける。根拠のない生成は基本的に抑制される仕組み
- [ ] S3 Vectors は**semantic search のみ対応（hybrid search非対応）**。ハイブリッド検索（キーワード＋ベクトル）が必要ならOpenSearch Serverless等を検討
- [ ] S3 Vectorsを使う場合、1ベクトルあたり**カスタムメタデータは1KBまで・35キーまで**という制限がある

## 片付け（⚠️最重要セクション）

このラボで作ったリソースを**全て**削除する。翌日に持ち越さない。

```bash
# 1. 作成した3つのKBのID・データソースIDを確認
aws bedrock-agent list-knowledge-bases --region us-east-1 \
  --query 'knowledgeBaseSummaries[].{id:knowledgeBaseId,name:name}'

# 2. 各KBについて、データソースを先に削除
aws bedrock-agent list-data-sources --region us-east-1 --knowledge-base-id <KB_ID> \
  --query 'dataSourceSummaries[].{id:dataSourceId,name:name}'
aws bedrock-agent delete-data-source --region us-east-1 \
  --knowledge-base-id <KB_ID> --data-source-id <DS_ID>

# 3. Knowledge Base本体を削除（3つとも）
aws bedrock-agent delete-knowledge-base --region us-east-1 --knowledge-base-id <KB_ID>
```

- Knowledge Base の削除ポリシーは既定で **Delete**（KB削除時にベクトルインデックス内のベクトルも自動削除される）。作成時に **Retain** に変更していた場合はベクトルストア側の削除が別途必要になるので、作成時の設定を思い出すこと
- Quick create で作った **S3 vector bucket** は、KB削除で中身が空になっていても**バケット自体は自動削除されない場合がある**。コンソール（S3 → Vector buckets）で確認し、残っていれば手動削除する

```bash
# S3 Vectors側の残存確認・削除（バケット名はコンソールのKB詳細 or CloudTrailで確認）
aws s3vectors list-vector-buckets --region us-east-1
aws s3vectors list-indexes --region us-east-1 --vector-bucket-name <BUCKET_NAME>
aws s3vectors delete-index --region us-east-1 --vector-bucket-name <BUCKET_NAME> --index-name <INDEX_NAME>
aws s3vectors delete-vector-bucket --region us-east-1 --vector-bucket-name <BUCKET_NAME>
```

```bash
# 元データのS3バケットも削除
aws s3 rm s3://kb-lab-<自分のアカウントID>-src --recursive
aws s3 rb s3://kb-lab-<自分のアカウントID>-src
```

- **もし誤って OpenSearch Serverless を選んで作成してしまっていた場合**は、上記のKB削除だけでは不十分なことがある。Bedrock コンソールで使ったコレクション名を控え、OpenSearch Serverlessコンソール（またはCLI `aws opensearchserverless list-collections`）で**コレクションが残っていないか必ず確認し、残っていれば即削除**する。放置すると時間課金が続く
- ローカルの `~/kb-lab/` ディレクトリはAWS課金に関係ないので削除は任意

最終確認として、AWS Cost Explorer で当日の Bedrock / S3 / OpenSearch Serverless の利用が0または想定内であることを翌日にもう一度チェックすると安心。

## 費用内訳の目安

| 項目 | 目安 |
|---|---|
| Titan Embeddings V2（3KB分の全文×3回同期） | 数セント |
| S3 Vectors ストレージ（数百KB、数日） | 1セント未満 |
| S3 Vectors PUT/クエリ（Part B/Cで数十回） | 数セント |
| Retrieve / RetrieveAndGenerate 呼び出し（Claude Haiku、数十回） | $0.5〜1程度 |
| S3標準ストレージ（元データ、数日） | 1セント未満 |
| **合計目安** | **$1〜3程度**（片付けを当日実施した場合） |

OpenSearch Serverlessを誤って使い、削除を翌日以降に持ち越すと、上記とは**桁が変わる**（1日あたり数百円〜千円規模）ので要注意。
