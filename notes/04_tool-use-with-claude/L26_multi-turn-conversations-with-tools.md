# L26 Multi-turn conversations with tools(ツールを使うマルチターン会話)

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション4「Claudeでのツール使用」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/multi-turn-conversations-with-tools (Claude Academy 日本語版、レッスン26/67、10分)
- 読んだ日: 2026-11-08
- 前のレッスン: 「ツール結果の送信」(L25)/ 次のレッスン: 「複数ターンの実装」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。
- 注意: 本文テキストのみを要約した。ページの図(マルチターンのツール呼び出しの流れ)は含まない。レッスン名は、サイトの表記が英語のまま。
- 注意(現行環境): ページの `chat` の例は、`temperature` と `stop_sequences` を持つ版で、`max_tokens` は 1000。YUTAの環境では、`temperature` は現行モデルで受け付けない(L06)、`max_tokens` は 1000 だと不足しうる(ex09 の経験)ため、そのまま写さず、ex11 までの `chat` を土台にする。
- 注意(ページの欠落): ページの `add_assistant_message` の更新コードは示されていない(ユーザー側のみ)。次のレッスンで出る可能性がある(未確認)。
- 注意(演習): このレッスン単独の演習は、まだ作っていない。ループの実装は L27 で行うため、L27 と合わせた演習を検討する。

## 1. このレッスンの目的
1つの質問に答えるために、Claude が複数のツールを順番に呼ぶ場面を理解し、それを処理する会話ループと、そのためのヘルパー関数の改修を理解する。

## 2. 課題(ページ)
- 複数のツールを使うアプリでは、Claude が1つの質問に答えるために、ツールを**順番に複数回**呼ぶ必要がある場面を処理しなければならない。
- 例: 「今日から103日後は何日ですか?」→ まず現在の日付を取得し、次にそれに103日を加算する。
- これにより、Claude が最終回答の前に複数のツールリクエストを行う、マルチターンの会話パターンが生まれる。アプリは、これを**自動的に**処理する必要がある。

## 3. マルチターンツールパターン(ページ)
| # | 内容 |
|---|---|
| 1 | ユーザーが「今日から103日後は何日ですか?」と尋ねる |
| 2 | Claude が `get_current_datetime` を要求する ToolUse ブロックで応答する |
| 3 | サーバーがその関数を呼び出し、結果を返す |
| 4 | Claude がさらに情報が必要と判断し、`add_duration_to_datetime` を要求する |
| 5 | サーバーがその関数を呼び出し、結果を返す |
| 6 | Claude が最終的な回答を出すのに十分な情報を得る |

## 4. 会話ループ(ページ)
Claude がツールを要求しなくなるまで続ける、会話ループが必要。
```python
def run_conversation(messages):
    while True:
        response = chat(messages)

        add_assistant_message(messages, response)

        # 疑似コード
        if response isn't asking for a tool:
            break

        tool_result_blocks = run_tools(response)
        add_user_message(messages, tool_result_blocks)

    return messages
```
- 流れ: ①Claude に送る → ②応答を履歴に足す → ③ツールを要求していなければ終了 → ④ツールを実行して結果ブロックを作る → ⑤結果をユーザーメッセージとして履歴に足し、①に戻る。
- 「ツールを要求しているか」の判定は、疑似コードのまま。実装は次のレッスン(未確認)。

## 5. ヘルパー関数のリファクタリング(ページ)
会話ループの前に、複数のメッセージブロックを扱えるよう、ヘルパーを更新する。

### メッセージハンドラーの更新
現在の `add_user_message`・`add_assistant_message` は、常にプレーンテキストを扱う前提。完全なメッセージオブジェクトを扱えるよう更新する。
```python
from anthropic.types import Message

def add_user_message(messages, message):
    user_message = {
        "role": "user",
        "content": message.content if isinstance(message, Message) else message
    }
    messages.append(user_message)
```
- 文字列、ブロックのリスト、完全な `Message` オブジェクトのどれでも渡せるようになる。

### チャット関数の更新
```python
def chat(messages, system=None, temperature=1.0, stop_sequences=[], tools=None):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
        "temperature": temperature,
        "stop_sequences": stop_sequences,
    }

    if tools:
        params["tools"] = tools

    if system:
        params["system"] = system

    message = client.messages.create(**params)
    return message
```
- ツールのリストを受け取り、**テキストだけでなく完全なメッセージを返す**。

### メッセージからのテキスト抽出
```python
def text_from_message(message):
    return "\n".join(
        [block.text for block in message.content if block.type == "text"]
    )
```
- メッセージ内のすべてのテキストブロックを見つけて結合する。最終応答をユーザーに表示するときに便利。

## 6. 主な改善点(ページ)
- 柔軟なメッセージ処理: ヘルパー関数が異なるメッセージ形式で動作する。
- チャットでのツールサポート: `chat` がツールスキーマを受け取って渡せる。
- 完全なメッセージの返却: テキストだけでなく完全なメッセージオブジェクトを取得でき、すべてのブロックが保持される。
- テキスト抽出ユーティリティ: 複雑なメッセージから、読みやすいテキストを取り出せる。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| multi-turn conversation with tools | ツールを使うマルチターン会話 |
| conversation loop | 会話ループ |
| pseudo-code | 疑似コード |
| refactor | リファクタリング(動作を変えずにコードを整理し直す) |
| `Message` object | `Message` オブジェクト(API の応答全体を表すもの) |
| `isinstance` | `isinstance`(値が、ある型かを調べる Python の関数) |
| text extraction | テキスト抽出 |

## 質疑(学習中の確認)
範囲: このレッスン(L26)の内容。

### 質問1: 会話ループの終了条件はどれか
出題: 「今日から103日後は何日ですか?」で、ツールが2つ(現在日時の取得、日付の加算)ある。会話ループの終了条件として最も適切なのは。
- A: ツールを2回呼んだら、必ず終了する。
- B: Claude の応答が、もうツールを要求しなくなったら終了する(`stop_reason` が `end_turn` になる)。
- C: 応答にテキストブロックが含まれたら、終了する。
- D: 10秒たったら、必ず終了する。
- YUTAの回答: B(「あるツールの実行結果を利用して別のツールを利用する可能性があるため。Claude が会話の終了と明示するまでループを続ける必要があるため」)
- 評価: 正解。理由も、ページの趣旨(結果に応じて次のツールを呼ぶ可能性がある)を押さえている。
  - 補足: 「Claude が会話の終了と明示する」は、正確には、**ツールを要求する応答(`stop_reason` が `tool_use`)かどうか**で判定する。Claude が文章で「終わります」と述べるかではなく、API の応答のフィールドで機械的に判断する。
  - B が正しい理由: 何回ツールが必要になるかは、質問によって変わり、Claude が結果を見て決める。回数を決め打ちせず、ツールを要求しなくなるまで続ける。
  - A: 今回は2回で足りても、別の質問では1回や3回以上になる。回数を固定すると、途中で打ち切るか、不要な呼び出しをする。
  - C: Claude はツールを呼ぶときにも、説明のテキストブロックを一緒に返すことがある(L24 のページの例)。テキストの有無では、終わりを判断できない。
  - D: 時間で切ると、処理の途中で止まり、不完全な回答になる。時間の上限は安全装置としては有用だが、終了条件の本体にはならない。
- 補足(設計の選択): 本体の終了条件は B で、それに加えて、安全装置として上限(回数や時間)を持つのが堅牢な設計。Claude が同じツールを繰り返し呼ぶなどで、ループが止まらない場合に備える(ex07 のリトライの上限と同じ考え方。ページには書かれていない)。

## 【補足】ページ外の内容(私の整理)
- **ex11 との関係**: ex11 で、`stop_reason` が `tool_use` の応答と `end_turn` の応答を実測した(L25 のノート参照)。会話ループの「ツールを要求しているか」の判定は、`stop_reason` を見る方法が、試験ガイドのエージェントループ(Task Statement 1.1 の付近)の書き方と一致する。ページの疑似コードは、判定方法を明示していない。
- **ループの設計上の注意**: `while True` は、終わらない可能性がある。上限回数を決めておく設計が必要になる。ガイドでも、ループの終了条件は `stop_reason` に基づく設計が問われると見ている(該当箇所は未確認)。
- **初出の補足**:
  - `isinstance(message, Message)`: 値 `message` が `Message` 型かを調べる。`Message` なら `.content` を取り出し、そうでなければ(文字列やブロックのリスト)そのまま使う。
  - `"\n".join([...])`: 文字列のリストを、改行でつないで1つの文字列にする。
- 公式ドキュメントでは、この `tool_use` のループを SDK が自動で行う **Tool Runner** が紹介されている(L25 のノート参照)。コースは、手動で作る方法を教えている。
- 試験ガイドとの関係: エージェントループ(Task Statement 1.1)の中核。ここから先の L27 で、ループの実装が出るため、重要度が高いと見ている(配点は、後で確認する)。
