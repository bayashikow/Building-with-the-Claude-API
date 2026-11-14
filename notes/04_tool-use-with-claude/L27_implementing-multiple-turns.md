# L27 複数ターンの実装(Implementing Multiple Turns)

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション4「Claudeでのツール使用」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/implementing-multiple-turns (Claude Academy 日本語版、レッスン27/67、15分)
- 読んだ日: 2026-11-08
- 前のレッスン: 「Multi-turn conversations with tools」(L26)/ 次のレッスン: 「複数のツールを使用する」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。
- 注意: 本文テキストのみを要約した。ページの図(複数ツール呼び出しの構造、ツール結果ブロック)は含まない。
- 注意(現行環境): ページの `chat` は `max_tokens` が 1000 で、`temperature` を渡す版(L26)。YUTAの環境では、ex11 までの `chat` を土台に直す。
- 注意(ページの省略): ページの `run_tools` は途中まで(`return` がない)。結果ブロックのリストを返す処理は、自分で足す想定。
- 注意(演習): このレッスン単独の演習は作らない。L26〜L28 を合わせた演習 ex12(`exercises/ex12_Multi-turn-tool-loop/multi-turn-tool-loop.ipynb`)を実施した(2026-11-14)。記録は、「実機での確認」にある。

## 1. このレッスンの目的
Claude がツールを要求しなくなるまで続ける会話ループを実装する。`stop_reason` による継続の判定、複数ツールの処理、エラー処理、ツールのルーティングを理解する。

## 2. ツールリクエストの検出(ページ)
- Claude がツールを使いたいかどうかの鍵は、応答メッセージの `stop_reason` フィールド。Claude がツールを呼ぶ必要があると判断すると、`"tool_use"` になる。
  ```python
  if response.stop_reason != "tool_use":
      break  # Claude is done, no more tools needed
  ```
- Claude がツールを要求しなくなったとき、利用者向けの最終応答が準備できたことを示す。

## 3. 会話ループ(ページ)
```python
def run_conversation(messages):
    while True:
        response = chat(messages, tools=[get_current_datetime_schema])
        add_assistant_message(messages, response)
        print(text_from_message(response))

        if response.stop_reason != "tool_use":
            break

        tool_results = run_tools(response)
        add_user_message(messages, tool_results)

    return messages
```
- Claude が、ツールを要求せずに最終回答を出すまで続く。

## 4. 複数のツール呼び出しの処理(ページ)
- Claude は1回の応答で複数のツールを要求できる。メッセージの `content` はブロックのリストで、各 ToolUse ブロックを個別に処理する必要がある。
- `run_tools` は、ToolUse ブロックを絞り込み、それぞれを処理する。
  ```python
  def run_tools(message):
      tool_requests = [
          block for block in message.content if block.type == "tool_use"
      ]
      tool_result_blocks = []

      for tool_request in tool_requests:
          # 各ツールリクエストを処理...
  ```

## 5. ツール結果ブロック(ページ)
- 各 ToolUse ブロックには、対応する `tool_result` ブロックで応答する。両者のつながりは、一致する ID で維持する。
  ```python
  tool_result_block = {
      "type": "tool_result",
      "tool_use_id": tool_request.id,
      "content": json.dumps(tool_output),
      "is_error": False
  }
  ```
- `content` は `json.dumps(...)` で文字列にしている。

## 6. エラー処理(ページ)
堅牢なツール実行には、エラーの処理が必要。**ツールが失敗した場合でも、Claude に結果ブロックを提供する**。
```python
try:
    tool_output = run_tool(tool_request.name, tool_request.input)
    tool_result_block = {
        "type": "tool_result",
        "tool_use_id": tool_request.id,
        "content": json.dumps(tool_output),
        "is_error": False
    }
except Exception as e:
    tool_result_block = {
        "type": "tool_result",
        "tool_use_id": tool_request.id,
        "content": f"Error: {e}",
        "is_error": True
    }
```

## 7. スケーラブルなツールルーティング(ページ)
複数のツールをサポートするには、ツール名から実装への対応を作る。
```python
def run_tool(tool_name, tool_input):
    if tool_name == "get_current_datetime":
        return get_current_datetime(**tool_input)
    elif tool_name == "another_tool":
        return another_tool(**tool_input)
    # 必要に応じてツールを追加
```
- コアとなる会話ロジックを変更せずに、新しいツールを簡単に足せる。

## 8. 完全なワークフロー(ページ)
1. 利用可能なツールとともに、ユーザーメッセージを Claude に送信する。
2. Claude が、テキストやツールリクエストで応答する。
3. 要求されたすべてのツールを実行し、結果ブロックを作る。
4. ツール結果を、ユーザーメッセージとして送り返す。
5. Claude が最終回答を出すまで繰り返す。

- 会話履歴は完全なコンテキストを維持し、Claude が以前のツール結果を基に、包括的な応答を出せる。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| `stop_reason` | 停止理由(応答が止まった理由。`"tool_use"` はツールの要求) |
| conversation loop | 会話ループ |
| tool routing | ツールのルーティング(ツール名から実装を選ぶ) |
| tool request | ツールリクエスト(ToolUse ブロック) |
| tool result block | ツール結果ブロック |
| `is_error` | エラーフラグ |
| list comprehension | リスト内包表記(条件に合う要素だけを集めるリストの書き方) |
| `json.dumps` | `json.dumps`(Python の値を JSON 文字列に変換する関数) |

## 質疑(学習中の確認)
範囲: このレッスン(L27)の内容。

### 質問1: ツールが例外で失敗したとき、どう処理するか
出題: 会話ループの実装で、ツールが例外を出して失敗した。最も適切な処理は。
- A: 例外をそのまま上に投げて、ループ全体を異常終了させる。
- B: `is_error: True` と、エラー内容の文字列を入れた `tool_result` を作り、Claude に返す。
- C: そのツール呼び出しは無かったことにして、`tool_result` を返さずに次の呼び出しに進む。
- D: 失敗したので、`content` を空の文字列にして `is_error: False` で返す。
- YUTAの回答: B(「ツールでエラーが発生したことを Claude に伝え、Claude の思考自体は止めないため」)
- 評価: 正解。理由も方向は合っている。より正確に言うと、「会話を続けられる状態を保つ」。さらに API のルール面でも重要で、ToolUse に対する `tool_result` が欠けると、次のリクエストが 400 エラーになる(L25 で確認した公式ドキュメントの記述)。
  - B が正しい理由: すべての ToolUse ブロックに対して、対応する `tool_result` を返す必要がある。失敗した場合も同じ。`is_error: True` とエラー内容を返せば、Claude は失敗を知り、引数を直して再試行する、別の方法を試す、利用者に事情を説明する、などの判断ができる。
  - A: 1つのツールの失敗で、会話全体が止まる。Claude に回復の機会を与えられない。
  - C: `tool_result` が欠けると、次のリクエストが API のルール違反でエラーになる(「tool_use ids were found without tool_result blocks immediately after」)。
  - D: 失敗なのに `is_error: False` で内容が空だと、Claude は「成功して空の結果が返った」と解釈する。失敗の事実が伝わらず、誤った回答につながる(L22 の「静かな失敗」と同じ問題)。
- 補足: エラーの文面は、「何が起きたか」「次に何を試すか」が分かるものが望ましい(L22・L25)。ページの `f"Error: {e}"` は最低限の形。ページの `try/except Exception` は、すべての例外を同じように扱う。実務では、再試行して直る失敗(一時的な通信エラー)と、直らない失敗(引数の誤り)を区別する設計もある。ページには書かれていない(私の整理)。

## 実機での確認(演習 ex12)
演習: `exercises/ex12_Multi-turn-tool-loop/multi-turn-tool-loop.ipynb`(L26〜L28 を合わせた演習。2026-11-14 実施)。モデルは `claude-haiku-5-5`。

### 内容
- 提供済み: ヘルパー関数、3つのツール(`get_current_datetime`、`add_duration_to_datetime`、`set_reminder`)の関数とスキーマ。
- 穴埋め(YUTAが実装):
  1. `run_tool`: 3つの分岐と、未登録のツール名を `ValueError` にする分岐。
  2. `run_tools`: ToolUse の絞り込み、`tool_result` の組み立て、`try/except` での `is_error`、結果リストの `return`。
  3. `run_conversation`: `stop_reason` による終了判定、履歴への追加、`max_turns` の安全装置(ページには無い、演習での設計)。
- 確認: TODO 1・2 は API なしの確認セルで通過。TODO 3 以降は、実際に API を呼んで実行。

### 観察した結果
- 依頼「医者の予約のリマインダーを設定してください。2050年1月1日から177日後です。」
  - ツールの順番: `add_duration_to_datetime`(`2050-01-01`、177 days)→ `set_reminder`。
  - `stop_reason`: `tool_use` が2回、`end_turn` が1回。
  - 日付計算の結果は `Monday, June 27, 2050`(ページの例と同じ)。
- `set_reminder` の `tool_result` は、`content` が `'null'`、`is_error: False`(関数が何も返さないため)。Claude はそれを成功と受け取った。時刻の指定がないのに、Claude が `09:00:00` を選んで設定し、最終応答で「午前9時に設定した」と伝えた。
- 未対応の単位(`fortnights`)を使わせる依頼: `is_error: True` の `tool_result`(`Error: Unsupported time unit: fortnights`)を受けた Claude は、**再試行せず、利用者にエラーを説明した**。使える単位を示し、`weeks` で 4 にする代案を挙げて、実行してよいかを尋ねた。
  - 公式ドキュメントでは、無効な呼び出しに対して Claude が補って再試行すると説明されている(L25 のノート)。今回は動きが違った。利用者が単位を明示していたためと推測するが、確かではない。1回の実行だけで、毎回同じとは言えない。
- 応答の `content` に `ThinkingBlock`(思考のブロック)が含まれていた。ループは `response` をそのまま履歴に足すので、一緒に保たれる。表示用の `show_history` は、`tool_use` / `tool_result` / `text` だけを表示する。

### つまずいた点と、わかったこと
- TODO 2 の確認で `TypeError: 'types.SimpleNamespace' object is not subscriptable`: ToolUse ブロックは、辞書ではなくオブジェクトで、`block.type`・`block.name`・`block.input`・`block.id` と、属性で読む。一方、`tool_result` は、自分で `{...}` と書く辞書なので、`block["key"]` で読む。
- 続けて `AssertionError`(`content` が `"Error:` から始まっていた): 失敗側の `content` を `json.dumps(f"Error: {e}")` と書いていたため、前後に `"` が付いた。`f"Error: {e}"` は、最初から文字列なので、`json.dumps` は要らない。`json.dumps` は、ツールの出力が辞書やリストのとき、文字列にするために使う。

### 質疑(演習後の確認)
質問1: `stop_reason` が `tool_use` / `end_turn` になった回数と、ツールの順番は。
- YUTAの回答: `tool_use` 2回、`end_turn` 1回。
- 評価: 回数は正解。ツールの順番(`add_duration_to_datetime` → `set_reminder`)の記載が抜けていた。後のツールの入力は、先のツールの結果から作られる。回数を決め打ちにできない理由(L26)と同じ。

質問2: `set_reminder` の `tool_result` の `content` は何か。成功と受け取ったか。どう直すか。
- YUTAの回答: `'null'`、成功とした、失敗した場合は `is_error` を `True` にする。
- 評価: 前半は正解。「どう直すか」は、半分。失敗時に `is_error: True` を返すことは、`run_tools` の `except` で、すでにできている(`set_reminder` が例外を出すことが前提)。ずれている点は、**成功時の戻り値が空**なこと。`{"status": "ok", "timestamp": ..., "content": ...}` のような確認できる値を返せば、Claude が設定内容を確かめられる。今回の実測でも、Claude が自分で選んだ `09:00:00` を、最終応答に述べていた。L22 の「静かな失敗」と近い話。

質問3: `is_error: True` を受けた Claude は、次に何をしたか。
- YUTAの回答: 利用者に説明した。
- 評価: 正解。代案(`weeks` で 4)も示した。上の「観察した結果」の補足のとおり、再試行する場合もある(確かでない)。

質問4: `max_turns` は、なぜ必要か。
- YUTAの回答: ツール失敗が連続した場合などに、多数の再試行を行う可能性がある。
- 評価: 方向は正解。「ツール失敗の連続」は、止まらなくなる場面の1つ。ほかに、結果の返し方の不具合で同じ依頼が繰り返される場合や、Claude が必要以上にツールを呼び続ける場合がある。上限が無いと、API の呼び出し回数、トークン、待ち時間が、止まるまで増え続ける。ex07 のリトライ上限と同じ考え方。

## 【補足】ページ外の内容(私の整理)
- **ex11 の実測との対応**: 最初の応答は `stop_reason == "tool_use"`、最終応答は `end_turn` だった(L25 のノート参照)。ページの `!= "tool_use"` で終了する判定は、`end_turn` 以外の終了理由(`max_tokens` など)も「終わり」と扱う。`max_tokens` で打ち切られた場合に、不完全な回答のまま終わる点には注意が要る(ex09 の経験。ページには書かれていない)。
- **ページのコードの注意点**:
  - `run_tools` は途中まで。`tool_result_blocks` を返す処理(`return`)を、自分で足す。
  - `json.dumps(tool_output)` は、ツールの出力が辞書やリストのときに必要。ex11 では、出力がすでに文字列だったので、そのまま渡せた。
  - `while True` は、終わらない可能性がある。上限回数などの安全装置を持つ設計が堅牢(L26)。ページには書かれていない。
- **初出の補足**:
  - `[block for block in message.content if block.type == "tool_use"]`: リスト内包表記。`content` のうち、`type` が `"tool_use"` のブロックだけを集める。
  - `try ... except Exception as e`: 処理中に例外が起きたら、`except` の中を実行する。`e` に例外の内容が入る。
- **試験ガイドとの関係**: ガイドの Task Statement 1.1 に、`stop_reason` が `"tool_use"` なら続け、`"end_turn"` なら終了する、というループ制御が書かれている。「ツール結果を会話履歴に足す」「モデルの判断でツールを選ばせる」といった設計も、同じ項目の近くにある。出題の内容は予想しない。
- 設計書で言えば、`run_tool` のルーティングは「機能の呼び出し表」で、新機能の追加時に、中心の処理を変えない設計。
- 公式ドキュメントでは、この `tool_use` のループを SDK が自動で行う **Tool Runner** が紹介されている(L25 のノート参照)。コースは、手動で作る方法を教えている。
