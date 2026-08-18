# LAB4: Bedrock Agents / Action Group（試験ドメインD2対応・90分・〜$1）

**狙い**: 問題集で頻出の「エージェントに社内APIを叩かせる」「traceでオーケストレーションをデバッグする」を「読んで知ってる」から「挙動を見た」に変える。Action Group + Lambda の最短構成を自分の手で組み、trace とセッション継続の仕組みを実機で確認する。

対応ドメイン: D2（生成AIソリューションの構築、配点26%）

想定所要時間: 90分　想定費用: 〜$1（Bedrockモデル呼び出し数回分＋Lambda無料枠内）

## ⚠️ 最初に: このラボが自分のアカウントで実行できるか確かめる

**Amazon Bedrock Agents（Classic）は 2026年7月30日にメンテナンスモードへ入り、新規顧客への提供を終了した。**
判定はアカウント単位で、AWS が「過去12か月に Bedrock Agents の利用実績があるか」で自動的に許可リストを決めている。**例外申請の窓口はない。**

| 自分のアカウント | このラボ |
|---|---|
| 過去12か月に Bedrock Agents の利用実績が**ある** | **Part A から通常どおり実施できる** |
| 実績が**ない**（多くの学習用アカウントはこちら） | Classic 手順は実行不可。**[付録: AgentCore で同じことをやる](#付録-agentcore-で同じことをやる新規アカウント向け)** へ進む |

**事前に可否だけを調べるAPIは無い。** `ListAgents` などの参照系は全アカウントで通るので、通っても許可リストの証明にはならない。
実際の可否は Part A の `CreateAgent`（コンソールの「エージェントを作成」でも同じ）で分かる。許可リスト外なら、次のエラーで弾かれる。

```
An error occurred (AccessDeniedException) when calling the CreateAgent operation:
Bedrock Agents is in Maintenance Mode. New agent creation is not available for
accounts without prior service usage.
```

これが出たら、Part A 以降は飛ばして付録へ。

> **試験対策としてどちらが要るか。** AIP-C01 の試験ガイド In-Scope に単独で載っているのは **Bedrock AgentCore** であって Bedrock Agents ではない。
> ただし出題論点としては Classic 側の語彙（Action Group・trace・Return of Control・`PrepareAgent`）が今も問われうるので、
> **手を動かせない場合でも Part A〜C の「確認ポイント」は読んでおく**こと。実機で触る対象は付録の AgentCore に寄せてよい。

## 前提条件

- 個人AWSアカウント、リージョンは **us-east-1**
- Bedrock コンソールでモデルアクセスが有効化済み（Claude Haiku系など、エージェントのオーケストレーションに使うモデル）
- IAMでロール作成・Lambda関数作成・Bedrock Agent作成ができる権限を持つIAMユーザー/ロールでlog in済み
- **本ラボのLambdaはダミー応答のみを返す**。実データ・実社名は一切使わない（都市名→ダミー天気情報を返すだけの例で進める）

## 確認ポイント（コラム）: Bedrock Agents Classic と AgentCore の住み分け

本ラボで扱う「Bedrock Agents」は、2023年11月ローンチの元祖 Bedrock Agents が改称された **Amazon Bedrock Agents Classic** である。AWS公式ドキュメントには以下の記載がある。

> Amazon Bedrock Agents (launched November 2023) is now Amazon Bedrock Agents Classic and will no longer be open to new customers starting on July 30, 2026.

**2026年7月30日にこれは発効した。** 以降、過去12か月に Bedrock Agents 利用実績のないアカウントでは `CreateAgent` と `InvokeInlineAgent` が `AccessDeniedException`（HTTP 403）になる。一方、`UpdateAgent` / `InvokeAgent` / `PrepareAgent` / 各種 Get・List・Delete など**既存エージェントの運用に必要なAPIは全アカウントで引き続き利用可能**で、既存エージェントが止まることはない。

公式ドキュメントで確認できる現況を整理すると、次のとおり。

| 論点 | 公式の記述 |
|---|---|
| 既存エージェントの停止 | **しない。** 通常どおり動作し、運用系APIも全アカウントで利用可能 |
| 制限されるAPI | **`CreateAgent` と `InvokeInlineAgent` のみ**（実績のないアカウントに限る） |
| 例外申請 | **なし。** 許可リストは過去12か月の利用実績からAWSが自動判定する |
| 移行期限 / EOL | **どちらも設定されていない。** Classic は既存顧客向けに維持される |
| 新機能・新モデル | **追加されない。** モデルカタログは 2026年7月30日時点で凍結され、以降の新モデルは AgentCore 側のみ |
| 名称変更の影響 | **なし。** API名前空間（`bedrock-agent`）・SDKクライアント・CloudFormation リソース型・IAM アクション接頭辞はすべて不変 |
| 料金 | Classic 自体への課金は元々なし。背後のモデル推論と関連リソースの課金のみ（変更なし） |

新規開発の推奨移行先は **Amazon Bedrock AgentCore**。公式は2つの経路を挙げている——設定ファイルでモデル・ツール・指示を宣言する **managed harness**（Classic のマネージド体験に最も近い）と、任意のフレームワーク（Strands / LangChain / OpenAI Agents SDK / Claude Agent SDK 等）を AgentCore Runtime に載せる **code-defined agents**。既存の Classic 設定を取り込む import 経路も AgentCore CLI にある。

本ラボは試験範囲である Classic の構成要素（Action Group・trace・InvokeAgent）を体感するのが目的なので、**許可リスト内のアカウントではこのまま従来型で進める**。そうでない場合は付録の AgentCore 版へ。

## Part A: エージェント作成とAction Group（コンソール操作）

### A-1. ダミーLambda関数を作成

Lambda コンソール → **関数の作成**

1. 「一から作成」を選択、関数名は任意（例: `agent-lab-weather-dummy`）
2. ランタイムは Python 3.13 系を選択
3. 以下のコードを貼り付ける。**実データは使わず、都市名に応じたダミー応答をハードコードで返すだけ**にする。

```python
import json

DUMMY_WEATHER = {
    "tokyo": {"condition": "晴れ", "temp_c": 28},
    "osaka": {"condition": "曇り", "temp_c": 26},
    "sapporo": {"condition": "雨", "temp_c": 19},
}

def lambda_handler(event, context):
    action_group = event["actionGroup"]
    function = event["function"]
    parameters = event.get("parameters", [])

    city = ""
    for p in parameters:
        if p["name"] == "city":
            city = p["value"].strip().lower()

    weather = DUMMY_WEATHER.get(city, {"condition": "不明", "temp_c": None})
    body_text = json.dumps(weather, ensure_ascii=False)

    function_response = {
        "actionGroup": action_group,
        "function": function,
        "functionResponse": {
            "responseBody": {
                "TEXT": {"body": body_text}
            }
        },
    }

    return {
        "messageVersion": "1.0",
        "response": function_response,
        "sessionAttributes": event.get("sessionAttributes", {}),
        "promptSessionAttributes": event.get("promptSessionAttributes", {}),
    }
```

4. **デプロイ** を押して保存する

このLambdaは「function detail方式」のイベント形式（`event["function"]` / `event["parameters"]`）を前提にしている。Action Groupの定義方式によって受け取るイベント形式が変わる点は後述の確認ポイントで扱う。

### A-2. エージェントを作成

Bedrock コンソール → 左ナビゲーション **Agents** → **Create Agent**

1. エージェント名（自動生成のままでも可）、説明は任意で入力し **Create**
2. 作成後は自動的に **Agent builder** 画面に遷移する
3. **Agent details** セクションで以下を設定:
   - **Agent resource role**: 「Create and use a new service role」を選択（Bedrockが自動でサービスロールを作成）
   - **Select model**: Claude Haiku系のモデルを選択
   - **Instructions for the Agent**: エージェントの役割を自然文で入力する。例:
     ```
     あなたは天気案内アシスタントです。ユーザーから都市名を聞かれたら get_weather アクションを使ってダミーの天気情報を取得し、簡潔に案内してください。
     ```
4. 一旦 **Save** して次に進む

### A-3. Action Groupを追加（function detail方式）

Agent builder画面の **Action groups** セクション → **Add**

1. **Action group details**: Name（例: `WeatherActions`）を入力
2. **Action group type**: **Define with function details** を選択
   - もう一方の選択肢「Define with API schemas」はOpenAPIスキーマ（JSON/YAML）を書く方式で、複数APIをまとめて定義できる代わりに準備に手間がかかる。単一の簡単なアクションを素早く定義したい今回のようなケースでは **function details方式の方が手早い**（OpenAPIスキーマの記述・バリデーションが不要なため）
3. **Action group invocation**: **Select an existing Lambda function** を選び、A-1で作成した関数とバージョン（`$LATEST` など）を選択
4. **Action group function** セクションで関数を定義:
   - Name: `get_weather`
   - Description: `指定した都市のダミー天気情報を返す`
   - **Parameters** → **Add parameter**:
     - Name: `city`
     - Description: `天気を知りたい都市名（例: tokyo, osaka, sapporo）`
     - Required: `True`
     - Type: `string`
5. **Add** を選択して保存

保存時に画面に注記が出る通り、**Bedrockからこのラムダを呼べるようにするには、Lambda側にリソースベースポリシーの付与が必要**（コンソールでこのフローに沿うと自動付与されることが多いが、念のためA-4で明示的に確認する）。

### A-4. Lambdaのリソースベースポリシーを確認・付与

Bedrockのサービスプリンシパル（`bedrock.amazonaws.com`）からLambdaを呼び出せるようにする、以下のリソースベースポリシーが必要（公式ドキュメントに明記されている構成）。

```bash
aws lambda add-permission \
  --function-name agent-lab-weather-dummy \
  --statement-id allow-bedrock-agent \
  --action lambda:InvokeFunction \
  --principal bedrock.amazonaws.com \
  --source-arn "arn:aws:bedrock:us-east-1:<ACCOUNT_ID>:agent/<AGENT_ID>" \
  --region us-east-1
```

- `--source-arn` にエージェントIDを条件として入れることで、**そのエージェントからの呼び出しに限定**できる（セキュリティのベストプラクティス）
- コンソールの「Select an existing Lambda function」フローで追加した場合、コンソールが自動でこの許可を付与していることが多いが、`aws lambda get-policy --function-name <関数名>` で実際に付与されているか確認しておくとよい

### A-5. Prepare Agent

Agent builder画面上部、または Test ウィンドウ内の **Prepare** を選択する。

- **エージェントやAction Groupの設定を変更したら、必ずPrepareしないとテストや呼び出しに反映されない**（作業用ドラフト = working draft を、テスト可能な状態にパッケージし直す操作）
- 画面の **Last prepared** 時刻を見て、最新の変更が反映されているか確認する癖をつける

### A-6. テストチャットで動作確認

エージェント詳細画面右側の **Test** ウィンドウで確認する。

1. 準備ができていなければ Test ウィンドウ内で **Prepare** を実行
2. 入力欄に以下を入力して **Run**:
   ```
   東京の天気を教えて
   ```
3. エージェントが `get_weather`（`city=tokyo` 相当のパラメータ）でLambdaを呼び出し、ダミーの天気情報を踏まえた応答を返すことを確認する

## Part B: trace でオーケストレーションを読む

Test ウィンドウで応答を生成中、または生成後に **Show trace** を選択する。

- trace はリアルタイムに更新され、各 **Step**（ステップ）ごとに、モデルへの入力プロンプト・推論設定・エージェントの推論過程・Action Group/Knowledge Baseの呼び出し結果を確認できる
- 各ステップを展開する矢印アイコンを選択すると詳細が開く

### trace種別（試験で問われるポイント）

trace は複数の種別（`Trace` オブジェクトの中の1ステップ）からなる。

- **PreProcessingTrace** — ユーザー入力を文脈化・カテゴリ分けし、有効な入力かを判定するステップ
- **OrchestrationTrace** — 入力を解釈し、Action Groupの呼び出しやKnowledge Baseへの問い合わせを行うステップ。以下の要素を含みうる:
  - **Rationale** — エージェントがなぜその行動を選んだかの推論テキスト
  - **InvocationInput** — Action Group/Knowledge Baseへの呼び出し内容（`invocationType` が `ACTION_GROUP` / `KNOWLEDGE_BASE` / `AGENT_COLLABORATOR` / `FINISH` のいずれか）。function detail方式のAction Groupなら `actionGroupName` / `function` / `parameters` が確認できる
  - **Observation** — 呼び出し結果（`actionGroupInvocationOutput` など）や、最終応答（`finalResponse`）
- **PostProcessingTrace** — オーケストレーションの最終出力を処理し、ユーザーへの返し方を決めるステップ
- **FailureTrace** — いずれかのステップが失敗した場合の失敗理由
- **GuardrailTrace** — Guardrailを関連付けている場合、介入有無（`GUARDRAIL_INTERVENED` / `NONE`）と評価詳細

今回のラボでは、`get_weather` を呼ぶと判断した **Rationale**、Lambdaへの呼び出しパラメータが載った **InvocationInput**（`actionGroupInvocationInput`）、Lambdaからの応答が載った **Observation**（`actionGroupInvocationOutput`）の3点セットを1ステップの中で確認できるはずである。

## Part C: boto3でinvoke_agentを叩く（セッション継続とtrace取得）

**重要な事実確認**: 2026年7月時点、`aws bedrock-agent-runtime` の AWS CLI サブコマンド一覧には **`invoke-agent` が存在しない**（`create-session` / `retrieve` / `retrieve-and-generate` などはあるが、エージェント本体の呼び出しコマンドはCLIに実装されていない）。公式ドキュメントの実装例もPython（boto3）で統一されている。そのため本パートはCLIではなく、**boto3を使った短いPythonスクリプト**で進める。

### C-1. 事前準備: エージェントIDとエイリアスIDを確認

```bash
aws bedrock-agent list-agents --region us-east-1 \
  --query 'agentSummaries[].{id:agentId,name:agentName}'
```

テスト用には `TSTALIASID`（working draftを指す固定のテストエイリアスID）がそのまま使える。恒久的なエイリアスを別途作る場合は `aws bedrock-agent create-agent-alias` を使うが、本ラボでは `TSTALIASID` で進めれば十分。

### C-2. invoke_agentスクリプト（2ターン、同一sessionIdで文脈継続を確認）

```python
# invoke_agent_lab.py
import uuid
import boto3

AGENT_ID = "<AGENT_ID>"
AGENT_ALIAS_ID = "TSTALIASID"
REGION = "us-east-1"

client = boto3.client("bedrock-agent-runtime", region_name=REGION)


def invoke(session_id: str, prompt: str, enable_trace: bool = True):
    response = client.invoke_agent(
        agentId=AGENT_ID,
        agentAliasId=AGENT_ALIAS_ID,
        sessionId=session_id,
        inputText=prompt,
        enableTrace=enable_trace,
    )

    completion = ""
    for event in response["completion"]:
        if "chunk" in event:
            completion += event["chunk"]["bytes"].decode()
        if "trace" in event:
            trace = event["trace"]["trace"]
            for step_type, payload in trace.items():
                print(f"--- trace: {step_type} ---")
                print(payload)

    print(f"\n[response] {completion}\n")
    return completion


if __name__ == "__main__":
    session_id = str(uuid.uuid4())

    # 1ターン目
    invoke(session_id, "東京の天気を教えて")

    # 2ターン目: 同じ session_id を使い回して文脈が継続することを確認
    invoke(session_id, "さっき聞いた都市の気温は何度だった？")
```

```bash
python invoke_agent_lab.py
```

- 同じ `sessionId` を2回のリクエストで使い回すことで、**エージェントが1ターン目の会話（東京の天気）を覚えたまま2ターン目に応答する**ことを確認する（セッション内の会話履歴を使った文脈維持）
- `enableTrace=True` にすると、レスポンスのイベントストリームに `chunk` だけでなく `trace` オブジェクトが混在して返ってくる。Part Bでコンソールに表示されていたtrace情報と同じ内容がJSON形式で取得できる
- レスポンスは **イベントストリーム**（`completion` がイテラブルなストリーム）で返る。各イベントは `chunk`（応答本文の断片。`bytes` フィールドをデコードする）を持ち、Knowledge Base連携時は `citations` が付くこともある。ストリーミングを有効にする場合は `streamingConfigurations` で `streamFinalResponse` を `True` にする（デフォルトは完全な応答が1つのchunkにまとまって返る）

### C-3. sessionState を使った補助情報の受け渡し（任意）

`sessionState` パラメータで `sessionAttributes` / `promptSessionAttributes` を渡すと、Lambda側の `event["sessionAttributes"]` として受け取れる（セッション全体で保持されるか、1ターンのみ保持されるかの違い）。Return of Control構成の場合は同じ `sessionState.returnControlInvocationResults` に実行結果を詰めて次の `invoke_agent` 呼び出しに渡す（後述）。

```python
response = client.invoke_agent(
    agentId=AGENT_ID,
    agentAliasId=AGENT_ALIAS_ID,
    sessionId=session_id,
    inputText="大阪の天気を教えて",
    enableTrace=True,
    sessionState={
        "sessionAttributes": {"preferred_unit": "celsius"},
    },
)
```

## Return of Control について（Lambdaを使わない代替方式）

今回のラボではLambda方式を使ったが、Action Groupの `actionGroupExecutor` を `{"customControl": "RETURN_CONTROL"}` に設定すると、**Lambdaを一切使わずクライアント側でaction実行を担う**モードになる。

- エージェントがactionを呼ぶべきと判断すると、Lambdaを呼ぶ代わりに `InvokeAgent` のレスポンスに `returnControl` フィールド（`invocationId` と `invocationInputs`）を含めて処理を打ち返す
- `invocationInputs` の各要素は `FunctionInvocationInput`（function detail方式の場合、`actionGroupName` / `function` / `parameters` を含む）または `ApiInvocationInput`（OpenAPIスキーマ方式の場合）
- クライアント側（アプリケーション側）で実際の処理を行い、同じ `invocationId` を添えて `sessionState.returnControlInvocationResults` に結果を詰め、再度 `invoke_agent` を呼ぶことでオーケストレーションを継続する
- ユースケース: Lambda実行時間の制約を超える長時間処理、Lambdaからアクセスしにくいネットワーク内のリソース呼び出し、複数APIの非同期・並列実行など

## 確認ポイント（チェックリスト）

- [ ] Action Groupの定義方式は **function details**（パラメータ定義のみ、素早く作れる）と **OpenAPIスキーマ**（JSON/YAML、複数API操作をまとめて定義）の2択で、**両方同時には指定できない**
- [ ] Action GroupのLambdaには、**Bedrockサービスプリンシパル（`bedrock.amazonaws.com`）からの`lambda:InvokeFunction`を許可するリソースベースポリシー**が必要（`aws lambda add-permission`で付与）。`SourceArn`条件でエージェントを限定するのがベストプラクティス
- [ ] エージェントやAction Groupの設定を変更した後は、**Prepare Agent（`PrepareAgent` API）をしないとテスト・呼び出しに最新の変更が反映されない**
- [ ] Lambdaが受け取るイベント・返すレスポンスの形式は、Action Groupの定義方式（function details か OpenAPIスキーマか）によって異なる（`event["function"]`系 vs `event["apiPath"]`系）
- [ ] **Return of Control**（`actionGroupExecutor: {"customControl": "RETURN_CONTROL"}`）は、Lambdaを使わずクライアント側でaction実行を担う方式。`InvokeAgent`レスポンスの`returnControl`（`invocationId`＋`invocationInputs`）を受け取り、実行結果を`sessionState.returnControlInvocationResults`に載せて次の`invoke_agent`で継続する
- [ ] `sessionId`を複数回の`invoke_agent`呼び出しで使い回すことで会話の文脈が継続する。`sessionState`で`sessionAttributes`/`promptSessionAttributes`を渡し、Lambda側の`event`として受け取れる
- [ ] `enableTrace=True`にすると、レスポンスのイベントストリームに`trace`オブジェクトが混在して返る。trace種別は**PreProcessingTrace / OrchestrationTrace / PostProcessingTrace / FailureTrace / GuardrailTrace**などがあり、OrchestrationTraceには`Rationale`（推論）・`InvocationInput`（呼び出し内容）・`Observation`（結果）が含まれる
- [ ] **AWS CLIには`bedrock-agent-runtime invoke-agent`コマンドが存在しない**（2026年7月時点）。エージェント呼び出しはboto3などのSDK経由で行う
- [ ] **Amazon Bedrock Agents（Classic）は2026年7月30日にメンテナンスモードへ入り、新規顧客への提供を終了した**。制限されるのは`CreateAgent`と`InvokeInlineAgent`だけで、対象は過去12か月に利用実績のないアカウント（`AccessDeniedException` / HTTP 403）。例外申請の窓口はない
- [ ] **既存エージェントは止まらない**。`UpdateAgent` / `InvokeAgent` / `PrepareAgent` / Get・List・Delete 系は全アカウントで利用可能。移行期限もEOL予定日も設定されていない。ただし**モデルカタログは2026年7月30日で凍結**され、新モデルは**Amazon Bedrock AgentCore**側にのみ追加される
- [ ] AgentCore への移行経路は2つ。設定で宣言する **managed harness**（Classic のマネージド体験に最も近い）と、任意フレームワークを載せる **code-defined agents**。Classic の Action Group は **AgentCore Gateway 経由の MCP ツール**に、Return of Control は **inline function ツール**に対応する

## 片付け

1. **エージェントの削除**（エイリアスも含めて削除される）
   ```bash
   aws bedrock-agent delete-agent --agent-id <AGENT_ID> --skip-resource-in-use-check --region us-east-1
   ```
   コンソールから行う場合は、エージェント詳細画面 → **Delete** を選択

2. **Lambda関数の削除**
   ```bash
   aws lambda delete-function --function-name agent-lab-weather-dummy --region us-east-1
   ```

3. **IAMロールの削除**（Bedrockが自動作成したサービスロール。コンソールのエージェント詳細画面またはIAMコンソールで削除前にロール名を確認しておく）
   ```bash
   aws iam list-attached-role-policies --role-name <AUTO_GENERATED_ROLE_NAME>
   # アタッチされているポリシーをデタッチしてからロールを削除
   aws iam delete-role --role-name <AUTO_GENERATED_ROLE_NAME>
   ```

4. **CloudWatch Logsのロググループ削除**（必要であれば）
   ```bash
   aws logs delete-log-group --log-group-name /aws/lambda/agent-lab-weather-dummy --region us-east-1
   ```

## 費用の目安

- Bedrockモデル呼び出し: テストチャット数回＋CLIスクリプト2ターン分。Haiku系モデルなら合計で数セント程度
- Lambda: 呼び出し回数が数回〜十数回程度なら無料枠内
- エージェント自体・Action Group自体に固定費用はなし（Bedrock Agentsの利用そのものへの追加課金はなく、背後のモデル呼び出し分のみ課金される）
- 合計で **$1未満** に収まる想定


---

## 付録: AgentCore で同じことをやる（新規アカウント向け）

Bedrock Agents Classic を作れないアカウント向けの代替経路。**「エージェントにツールを持たせ、ツール呼び出しの様子をトレースで追う」というLAB4の狙いはそのまま**で、土台を AgentCore の managed harness に置き換える。

> **本付録の裏取りについて。** 手順・コマンドは [AgentCore CLI 入門](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-get-started-cli.html)・[harness 入門](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-get-started.html)・[harness のツール](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-tools.html) の記述に沿って構成している。
> **ラボ本体（Part A〜C）と違い、通しでの実機確認は行っていない。** 画面やCLIの挙動が違っていたら Issue で知らせてほしい。

### 用語の対応表（Classic → AgentCore）

| Bedrock Agents Classic | AgentCore での相当物 |
|---|---|
| マネージドなオーケストレーションループ | harness が標準で提供 |
| Action Group（OpenAPI/関数スキーマ＋Lambda実行体） | **AgentCore Gateway** が REST API や Lambda を MCP ツールとして公開 |
| Return of Control（クライアント側で実行） | **inline function ツール**（harness が停止し `tool_use` をクライアントへ返す） |
| Knowledge Base をエージェント設定に紐づけ | Gateway 経由の KB 連携、またはコード側の retrieval ツール |
| trace UI / trace API | AgentCore observability（全アクションの永続トレース） |
| `AMAZON.CodeInterpreter` | AgentCore Code Interpreter ツール |
| セッション・メモリ設定 | AgentCore Memory（短期・長期、戦略を選択） |
| ステージ別のプロンプトオーバーライド | **直接の相当物なし**（system prompt ＋自前スクリプトで近似する） |
| マルチエージェント協調のルーティング | **限定的**（agent-as-tool は可能。ルーティング型は自前コードが要る） |

出典: [Amazon Bedrock Agents Classic maintenance mode](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html)

**この表の右2行が試験でも実務でも効く。** 「Classic からの移行で何が失われるか」を問われたら、**ステージ別プロンプトオーバーライドとルーティング型マルチエージェント**が答えの中心になる。

### 前提条件

- **Node.js 20 以降**（AgentCore CLI は npm パッケージ）。`node --version` で確認
- **Python 3.10 以降**（エージェントコードを書く場合）
- AWS 認証情報が設定済みで、**AgentCore がサポートするリージョン**であること（[対応リージョン一覧](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html)を先に確認する。ラボ本体の us-east-1 と同じとは限らない）
- AgentCore API を呼ぶ権限と、デプロイ時に CDK bootstrap ロールを assume できる IAM 権限（[必要な権限](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/security-iam.html)）

### 手順

**1. CLI をインストール**

```bash
npm install -g @aws/agentcore
agentcore --version
```

**2. harness プロジェクトを作る**

```bash
agentcore create
```

対話ウィザードで、プロジェクト種別に **Harness**（設定ベースのマネージドループ。フレームワークのコードを書かない）を選ぶ。モデルプロバイダ・メモリ・環境を順に選択して確定する。非対話で作るなら次のとおり。

```bash
agentcore create --name weatherlab --model-provider bedrock
```

**3. ツールを1つ持たせる（Part A の Lambda に相当）**

LAB4 の Lambda は「都市名を受けてダミー天気を返すだけ」だった。AgentCore では、**同じことを Lambda もIAMロールも作らずに** inline function ツールで再現できる。ツールの実体はクライアント側（手元）で動き、harness は呼び出しを打ち返してくる——**Classic の Return of Control と同じ形**である。

```bash
agentcore add tool --harness weatherlab --type inline_function \
  --name get_weather \
  --description "Get the current weather for a city." \
  --input-schema '{"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}'
```

> **Lambda を本当に繋ぎたい場合**は inline function ではなく **AgentCore Gateway** を立て、Lambda をターゲットに登録して MCP ツールとして公開し、`agentcore add tool --type agentcore_gateway --gateway-arn <ARN>` で harness に取り付ける。これが Action Group の正統な移行先だが、Gateway の作成手順は本付録の範囲外なので [Gateway のドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)を見ること。

**4. ローカルで動かす（任意）**

```bash
agentcore dev
```

依存をインストールしてローカルサーバを起動し、ブラウザで **agent inspector** が開く。チャットしながらトレースをその場で見られる。**Part C で trace を読んだのと同じ観察が、ここで一番安く済む。**

**5. デプロイして呼ぶ**

```bash
agentcore deploy
agentcore invoke --harness weatherlab \
  --session-id "$(uuidgen)" \
  "東京の天気は？"
```

初回デプロイは CDK の bootstrap が走るので数分かかる。**同じ `--session-id` を使い回すと会話が継続する**——Classic の `sessionId` と同じ考え方（ただし `runtimeSessionId` は33文字以上が要求されるので UUID を使う）。

エージェントが `get_weather` を呼ぶと、非対話モードでは `stopReason: "tool_use"` で戻ってくる。手元で結果を作り、続けて invoke に載せて返すとオーケストレーションが再開する。**この往復こそが Return of Control の実物**なので、一度は手で回しておく。

**6. トレースとログを見る（Part C 相当）**

```bash
agentcore logs --since 30m
agentcore traces list
agentcore traces get <trace-id>
```

**7. 片付け（必ずやる）**

```bash
agentcore remove all
agentcore deploy
```

`remove all` は設定を空にするだけ。**続けて `agentcore deploy` を打って初めてアカウント上のリソースが削除される。** ここを忘れると課金が続くので、`agentcore status` で残骸がないことまで確認する。

### 費用について

**Classic と違い、AgentCore は従量課金の対象が増える。** harness そのものへの追加課金はないが、runtime・memory・gateway といった各機能の消費に応じて課金される。ラボ規模（数回の invoke と即日削除）なら少額に収まる想定だが、**本ラボ集の「〜$1」は Classic 手順の見積もりであって、AgentCore 版で検証した数字ではない**。実施前に [AgentCore の料金ページ](https://aws.amazon.com/bedrock/agentcore/pricing/)を必ず見て、LAB0 の予算アラートを効かせた状態で始めること。
