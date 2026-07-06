# LAB4: Bedrock Agents / Action Group（試験ドメインD2対応・90分・〜$1）

**狙い**: 問題集で頻出の「エージェントに社内APIを叩かせる」「traceでオーケストレーションをデバッグする」を「読んで知ってる」から「挙動を見た」に変える。Action Group + Lambda の最短構成を自分の手で組み、trace とセッション継続の仕組みを実機で確認する。

対応ドメイン: D2（生成AIソリューションの構築、配点26%）

想定所要時間: 90分　想定費用: 〜$1（Bedrockモデル呼び出し数回分＋Lambda無料枠内）

## 前提条件

- 個人AWSアカウント、リージョンは **us-east-1**
- Bedrock コンソールでモデルアクセスが有効化済み（Claude Haiku系など、エージェントのオーケストレーションに使うモデル）
- IAMでロール作成・Lambda関数作成・Bedrock Agent作成ができる権限を持つIAMユーザー/ロールでlog in済み
- **本ラボのLambdaはダミー応答のみを返す**。実データ・実社名は一切使わない（都市名→ダミー天気情報を返すだけの例で進める）

## 確認ポイント（コラム）: Bedrock Agents Classic と AgentCore の住み分け

本ラボで扱う「Bedrock Agents」は、2026年7月時点で **Amazon Bedrock Agents Classic** と呼ばれている（2023年11月ローンチの元祖Bedrock Agentsが改称されたもの）。AWS公式ドキュメントには以下の記載がある。

> Amazon Bedrock Agents (launched November 2023) is now Amazon Bedrock Agents Classic and will no longer be open to new customers starting on July 30, 2026.

つまり **2026/7/30以降、過去12か月にBedrock Agents利用実績のないアカウントでは新規の `CreateAgent` がAccessDeniedExceptionになる**（既存の利用実績があるアカウントは影響なし）。`UpdateAgent` / `InvokeAgent` / 各種GetAPIなど既存エージェントの運用に必要なAPIは引き続き全アカウントで利用可能。新規開発の推奨移行先は **Amazon Bedrock AgentCore**（Runtime / Memory / Gateway / Identity 等のモジュール群からなる、フレームワーク非依存のエージェント基盤）。

本ラボは試験範囲であるAgents Classicの構成要素（Action Group・trace・InvokeAgent）を体感するのが目的なので、このまま従来型で進める。既に自分のアカウントでBedrock Agentsを使った実績があれば影響なく進められるはずだが、**アカウントが新規で7/30以降に本ラボを行う場合はCreateAgentがブロックされる可能性がある**点は覚えておく（試験にも「Classic は新規受付終了、AgentCoreへの移行が推奨」という論点で出うる）。

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
- [ ] **Amazon Bedrock Agents（Classic）は2026年7月30日以降、新規顧客（過去12か月に利用実績のないアカウント）向けの`CreateAgent`を受け付けなくなる**。既存の利用実績があるアカウントおよび既存エージェントの運用（Update/Invoke等）には影響なし。新規開発は**Amazon Bedrock AgentCore**への移行が推奨されている

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
