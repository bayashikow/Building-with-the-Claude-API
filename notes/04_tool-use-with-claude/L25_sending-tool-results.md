# L25 ツール結果の送信(Sending Tool Results)

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション4「Claudeでのツール使用」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/sending-tool-results (Claude Academy 日本語版、レッスン25/67、10分)
- 読んだ日: 2026-11-08
- 前のレッスン: 「メッセージブロックの処理」(L24)/ 次のレッスン: 「Multi-turn conversations with tools」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。
- 注意: 本文テキストのみを要約した。ページの図(ツール結果ブロックの構造、複数呼び出し)は含まない。
- 演習: L22〜L25 を合わせた ex11 を実施した(2026-11-08)。記録は「実機での確認」を参照。

## 1. このレッスンの目的
Claude のツール呼び出しの依頼に対して、関数を実行し、その結果を送り返して、ツール使用の流れを完成させる。

## 2. ツール関数の実行(ページ)
- Claude が ToolUse ブロックで応答したら、入力パラメータを取り出して関数を呼ぶ。
  ```python
  response.content[1].input
  ```
  これで、Claude が関数に渡したい引数の**辞書**が得られる。
- 関数はキーワード引数を期待するため、辞書をそのまま渡さず、Python のアンパック構文 `**` で展開する。
  ```python
  get_current_datetime(**response.content[1].input)
  ```

## 3. ツール結果ブロック(ページ)
- ツール関数を実行した後、**ツール結果ブロック**で、結果を Claude に送り返す。このブロックは**ユーザーメッセージの内部**に置く。
| プロパティ | 内容 |
|---|---|
| `tool_use_id` | この結果が対応する ToolUse ブロックの `id` と一致させる |
| `content` | ツールを実行した出力。文字列としてシリアライズする |
| `is_error` | エラーが起きたときに True |

## 4. 複数のツール呼び出し(ページ)
- Claude は1回の応答で、複数のツール呼び出しを依頼できる。例: 「10 + 10はいくつで、30 + 30はいくつですか?」→ 2つの別々の ToolUse ブロック。
- 各呼び出しには一意の ID が付く。結果を返すときは、ID を一致させる。結果が異なる順序で届いても、Claude は、どの結果がどの依頼か把握できる。

## 5. フォローアップリクエストの構築(ページ)
```python
messages.append({
    "role": "user",
    "content": [{
        "type": "tool_result",
        "tool_use_id": response.content[1].id,
        "content": "15:04:22",
        "is_error": False
    }]
})
```
完全な会話履歴は、次の3つを含む。
1. 元のユーザーメッセージ
2. ToolUse ブロックを含むアシスタントメッセージ
3. ツール結果ブロックを含むユーザーメッセージ

## 6. 最終リクエストの送信(ページ)
- フォローアップを送るとき、Claude に別のツール呼び出しを期待していなくても、**ツールスキーマ(`tools`)を含める必要がある**。会話履歴内のツールへの参照を理解するのに、スキーマが要る。
  ```python
  client.messages.create(
      model=model,
      max_tokens=1000,
      messages=messages,
      tools=[get_current_datetime_schema]
  )
  ```
- Claude は、ツール結果をユーザー向けの自然な応答に組み込んだ、最終メッセージで応答する。これでツール使用のワークフローは完了。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| tool result block (`tool_result`) | ツール結果ブロック |
| `tool_use_id` | ツール使用 ID(対応する ToolUse ブロックの ID) |
| `is_error` | エラーフラグ |
| unpacking (`**`) | アンパック(辞書をキーワード引数に展開する構文) |
| keyword argument | キーワード引数 |
| serialize | シリアライズ(文字列などの形に変換する) |
| follow-up request | フォローアップリクエスト(結果を返す次の呼び出し) |

## 質疑(学習中の確認)
範囲: このレッスン(L25)の内容。

### 質問1: 2つの ToolUse に対する結果の返し方はどれか
出題: Claude が、ID が `toolu_A` と `toolu_B` の2つの ToolUse ブロックを返した。結果の返し方として最も適切なのは。
- A: 2つの結果を1つの文字列にまとめ、`tool_use_id` には `toolu_A` だけを指定する。
- B: 結果ごとに `tool_result` ブロックを作り、それぞれの `tool_use_id` に対応する ID を指定して、ユーザーメッセージに入れる。
- C: 結果を、アシスタントメッセージとして履歴に追加する。
- D: ID は不要なので、結果の文字列だけを、ユーザーメッセージとして送る。
- YUTAの回答: B(「ID によりどの結果がどのツール実行によるものかを判断する必要があるため」)
- 評価: 正解。理由も、ページの趣旨(ID で結果と依頼を対応づける)を押さえている。
  - A: 2つの結果を1つにまとめ、ID を1つしか指定しないと、`toolu_B` の依頼に対する結果が存在しない形になる。どちらの結果か区別もつかない。
  - C: ツール結果は、アシスタントメッセージではなく、**ユーザーメッセージ**に入れる(ページの記述)。アシスタントメッセージは Claude 自身の発言を表す。
  - D: ID が無いと、結果と依頼を結びつけられない。複数の呼び出しがあると、取り違えの原因になり、API 側でもエラーになるおそれがある。
- 補足: 「アシスタント=Claude の発言、ユーザー=アプリ側からの入力」という役割分担は、ここでも変わらない。ツールの結果は、アプリ側が Claude に渡す入力なので、`role` は `user`。

### 質問2(振り返り1〜4): ex11 の振り返り
- 1. 最初の応答の `stop_reason` と `content`
  - YUTAの回答: 「tool_use」
  - 評価: 正解。`content` は、ToolUse ブロックのみ(テキストブロックなし)だった。ページの例(テキスト+ToolUse)と違う。位置を決め打ちせず、種類で探す書き方が必要だった実例。
- 2. Claude が渡した引数
  - YUTAの回答: 「'date_format': '%H:%M:%S'」
  - 評価: 正解。依頼した「HH:MM:SS」を、Claude が `strftime` の書式に変換して渡した。
- 3. `tools` を外した最終リクエスト(任意の実験)
  - YUTAの回答: 「エラーになりませんでした」
  - 評価: 観察として有効。ページは「`tools` を含める必要がある」と書くが、今回の環境では成功した。扱いは下の「実機での確認」を参照。
- 4. `tool_result` のメッセージの `role` が `user` の理由
  - YUTAの回答: 「user のサーバから返された結果だから」
  - 評価: 概ね正解。補足: `role: user` は、利用者本人のサーバーではなく、**Claude に入力を渡す側**(自分のアプリ)を表す。公式ドキュメントも、`user` メッセージはクライアント側の内容と `tool_result` を含み、`assistant` メッセージは AI が生成した内容と `tool_use` を含む、と説明している(要約)。

## 実機での確認(YUTAの環境、2026-11-08)
- ノートブック: `exercises\ex11_Tool-use-basics\tool-use-basics.ipynb`(ex10 のセットアップを引き継ぎ、TODO とヒントを入れた課題)。新しく覚えること: ツール使用の往復を1回、通して動かす。API 呼び出しは2回。モデルは `claude-haiku-5-5`。
- TODO 1〜3 は、すべて正しく書けた。
  - TODO 1: 入力検証つきの `get_current_datetime`。空の `date_format` で `ValueError`(メッセージは「date_format無しは受け付けられません」)。確認セルは「OK」。
  - TODO 2: スキーマ(`name`・`description`・`input_schema`)。確認セルは「OK」。
  - TODO 3: `tools` 付きの最初の呼び出し。`block.type == "tool_use"` で ToolUse ブロックを探した。
- 最初の応答の実測: `stop_reason` は `tool_use`、`content` は `['tool_use']`(ToolUse ブロックのみ)。ブロックは `ToolUseBlock(id=..., caller=DirectCaller(type='direct'), input={'date_format': '%H:%M:%S'}, name='get_current_datetime', type='tool_use', toolset_name=None)`。`caller` と `toolset_name` の意味は、未確認(ドキュメントに、コンピューター操作・ブラウザ操作のツール群で `toolset_name` を使う記述があった。今回は関係しない)。
- TODO 4 でのエラー:
  ```
  AttributeError: 'list' object has no attribute 'tool_use'
  ---> "tool_use_id": response.content.tool_use.id,
  ```
  - 読み解き: `response.content` は list(ブロックの並び)で、`.tool_use` という属性は無い。ToolUse ブロックは、TODO 3 で `tool_use_block` に取り出してある。
  - 同じセルで、`"content": "15:04:22"`(ページの例の値)が残っていた。エラーにならないが、実際の時刻と違う値を返してしまうので、`tool_output` に直した。
  - 注意: このセルは、エラーの前に、アシスタントメッセージを `messages` に追加済みだった。そのまま再実行すると、同じメッセージが二重に追加され、後で 400 エラーになる。TODO 3 から実行し直して、履歴をリセットした(`messages = []`)。
- 修正: `response.content[0].id` に直して動いた。ただ、位置の決め打ちなので、`tool_use_block.id` に再修正した。
- 最終応答の実測: `stop_reason` は `end_turn`、`content` は `['text']`。本文は「現在の時刻は **15:34:21** です。」で、関数の出力(15:34:21)と一致した。

  | 場面 | `stop_reason` | `content` |
  |---|---|---|
  | 最初の応答 | `tool_use` | ToolUse ブロックのみ |
  | 最終応答 | `end_turn` | テキストブロックのみ |

- この対応は、試験ガイドのエージェントループ(`tool_use` なら続け、`end_turn` で終える。Task Statement 1.1 の付近)の、最小の実例。
- `tools` を外した最終リクエスト(任意の実験): エラーにならなかった。
  - 事実: この環境(`claude-haiku-5-5`、2026-11-08)では、`tools` なしの最終リクエストが成功した。ページの「`tools` を含める必要がある」と食い違う。
  - 確認できていない点: 常にそうなのか、モデルや API の版で変わるのか。公式ドキュメント(Handle tool calls、Troubleshooting tool use)には、この点の明示を見つけられなかった。実験で `tools` の行を実際に外して実行したかは、保存済みのノートブック(元に戻した状態)からは確認できない。
  - 方針: 動いても、`tools` は含めたままにする。ドキュメントは、サーバーツールを使う再開時には、`tools` 配列を同じに保つよう案内している。

## 【補足】ページ外の内容(私の整理)
### 公式ドキュメントでの確認(確認日 2026-11-08)
出典: [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)(Claude Platform Docs)。以下は私による要約。
- ツール使用の応答は、`stop_reason` が `tool_use` で、1つ以上の `tool_use` ブロック(`id`・`name`・`input`)を含む。ページの説明と一致する。
- 結果を返すメッセージは、`role` が `user`。`tool_result` ブロックに、`tool_use_id`、`content`(省略可)、`is_error`(省略可)を持たせる。
- `content` は、文字列のほか、ブロックのリスト(テキスト・画像・文書など)も可。ページは「文字列」と書くが、ドキュメントでは、より広く許される。
- **書式の要件**(ドキュメントの Important):
  - ツール結果ブロックは、対応する ToolUse ブロックの**直後**のメッセージに置く。間に他のメッセージを入れられない。
  - ツール結果を含むユーザーメッセージでは、`tool_result` ブロックを content 配列の**先頭**に置く。テキストは、すべての結果の**後ろ**に置く。テキストを前に置くと、400 エラーになる例が示されている。
  - 満たさないと、「tool_use ids were found without tool_result blocks immediately after」のようなエラーが出る。
- **エラーの返し方**: ツール実行が失敗したら、エラーの内容を `content` に入れ、`is_error: true` にして返す。`failed` のような一般的な文より、「何が起きたか」「次に何を試すべきか」を含めるほうがよいとされる(L22 の「意味のあるエラーメッセージ」と同じ考え方)。
- 引数が足りないなどの無効な呼び出しには、エラーの `tool_result` を返すと、Claude が補って再試行する。ドキュメントは、「2〜3回、修正して再試行したのち、利用者に謝る」と説明する。
- 無効な呼び出しをなくすには、ツール定義に `strict: true`(strict tool use)を付ける方法が紹介されている(別ページ。今回は未確認)。
- ツール結果には、Web ページやメールなど、外部の内容が入りうる。信頼できない内容として扱い、`tool_result` ブロックの中に保つようにという警告がある(プロンプトインジェクション対策)。
- SDK の **Tool Runner** が、この手動の処理(`tool_use` のループ、結果の整形、再試行)を自動で行う。手動の処理は、実行を細かく制御したいときに使う、と説明されている。

### 未確認の点
- 「複数の `tool_result` を1つのユーザーメッセージにまとめる」規則は、このページには明示がない。並列ツール使用のページ(Parallel tool use)に書かれているはずだが、まだ読んでいない(未確認)。
- 「フォローアップに `tools` を含める必要」(ページの記述)は、このドキュメントページでは確認していない。 実機では、`tools` なしでも成功した(「実機での確認」を参照)。

### ページのコードの注意点
- `response.content[1]` は、「2番目のブロックが ToolUse」と決め打ち。テキストブロックが無い、または複数ある場合に、位置が変わる。`block.type == "tool_use"` で探すほうが安全(L24 と同じ、種類で判別する書き方)。

### 設計上の読み取り
- L20 の4ステップとの対応: 初期リクエスト(L24)→ ツールリクエスト(L24)→ データ取得=関数の実行(L22・L25)→ 最終応答(L25)。
- ID は、設計書で言えば、依頼と回答を結びつける「伝票番号」。複数件を並行して依頼しても、取り違えを防げる。
- 試験ガイドとの関係: エージェントループ(`stop_reason` が `"tool_use"` の間は、ツールを実行して結果を返し、`"end_turn"` で終了する。Task Statement 1.1 の付近)の中身が、この手順にあたる。
