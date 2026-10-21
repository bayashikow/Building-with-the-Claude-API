# L08 構造化データ(Structured Data)

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション1「APIを使ったClaudeへのアクセス」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/structured-data (Claude Academy 日本語版、レッスン8/67、15分)
- 読んだ日: 2026-10-21
- 前のレッスン: 「レスポンスストリーミング」(L07)/ 次のレッスン: 「APIを使用したClaudeへのアクセスに関するクイズ」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。「実機での確認」は、YUTAが自分のノートブックで動かした結果。
- 注意: 本文テキストのみを要約した。

> **重要: このレッスンの教材の方法は、一定のモデルバージョン以降では使えません。**
> - 教材が示す方法は、「アシスタントメッセージのプリフィル(prefill)+ ストップシーケンス」です。
> - 公式ドキュメントでは、**Claude 4.6 以降のモデルは、最後のアシスタントのターンにプリフィルを置くと、400 エラーになる**、と書かれています(確認日 2026-10-21。下の【補足】に出典)。Claude Sonnet 5.5 でも、同様に拒否されると、Sonnet 5.5 の移行ガイドに書かれています。Claude Haiku 4.5 以前のモデルは、プリフィルを受け付けます。
> - 現行の方法は、**構造化出力(Structured outputs)**です。`create()` の `output_config.format` に、JSON スキーマを渡します。詳しくは、「現行の方法」の節と、「実機での確認」を参照してください。
> - YUTAが使っている `claude-haiku-5-5` で、教材の方法(プリフィル)を、実際に動かして確かめたかは、まだです(未確認)。構造化出力が、`claude-haiku-5-5` で動くことは、実機で確認しました。

## 1. このレッスンの目的(ページ)
JSON、Python コード、箇条書きなどの構造化データを、Claude に生成させるとき、Claude は、役に立とうとして、内容の周りに、説明文を足したがる。説明のない、生のデータだけを取り出す方法を学ぶ。

## 2. 問題(ページ)
- 例: AWS EventBridge のルールを生成する Web アプリで、ユーザーが生成を押したら、すぐ使えるクリーンな JSON が欲しい。
- Claude が、説明文つきの、マークダウンのコードブロックで JSON を返すと、ユーザーは、JSON の部分だけを、手で選ばなければならない。
- 既定では、```json で包まれた JSON と、その下に、説明の文が付くことがある。JSON 自体は正しいが、生のデータとして、そのまま使いにくい。

## 3. 教材の解決策: アシスタントメッセージのプリフィル + ストップシーケンス(ページ)
> 注意: この方法は、上の「重要」のとおり、一定のモデルバージョン以降では使えません。

```python
messages = []

add_user_message(messages, "Generate a very short event bridge rule as json")
add_assistant_message(messages, "```json")

text = chat(messages, stop_sequences=["```"])
```
仕組み:
1. ユーザーメッセージが、何を生成するかを伝える。
2. プリフィルされたアシスタントメッセージ(```json)により、Claude は、すでにコードブロックを始めたと思い込む。
3. Claude は、続けて、JSON の内容だけを書く。
4. Claude が、``` で、コードブロックを閉じようとすると、ストップシーケンスが、生成を終わらせる。

結果: 余分な書式のない、クリーンな JSON。

後処理(ページ):
```python
import json

clean_json = json.loads(text.strip())
```
- 余分な改行が付くことがあるので、`strip()` で取ってから、パースする。

応用先(ページ): Python コードのスニペット、箇条書き、CSV など、解説なしで構造化データが欲しい場面全般。重要なのは、Claude が、自然に、内容を何で包みたがるかを見極めて、それを、プリフィルとストップシーケンスに使うこと。

## 現行の方法: 構造化出力(Structured outputs)【補足: ページ外。公式ドキュメントで確認】
出典: https://platform.claude.com/docs/en/build-with-claude/structured-outputs (確認日 2026-10-21)

### 2つの機能
| 機能 | 内容 |
|---|---|
| JSON 出力(`output_config.format`) | 応答の形式を、JSON スキーマで制限する |
| 厳密なツール使用(`strict: true`) | ツール名と入力の、スキーマ検証を保証する |

### 書き方(`create()` の場合。ドキュメントの例を基にした形)
```python
response = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=messages,
    output_config={
        "format": {
            "type": "json_schema",
            "schema": schema,
        }
    },
)
```
- `output_config` → `format` → `type` と `schema` の、3段。
- `format` の `type` は `"json_schema"`(形式の種類)。`schema` の中の `type` は、`"object"`、`"array"`、`"string"` など(JSON のデータ型)。2つの `type` は、階層も意味も、別。
- 取り出し: `content` の中から、`type` が `"text"` のブロックを探して、`.text` を、`json.loads()` で辞書にする。ドキュメントの例も、`content[0]` を決め打ちにしていない。
- 推奨(ドキュメント): `client.messages.parse()` に、Pydantic のモデルを渡す。結果は `response.parsed_output` で取れる(今回は、使っていない)。
- 以前の書き方の `output_format` は、`output_config.format` に移った。ベータ用のヘッダーは、不要になった。

### 対応モデル
- Haiku 4.5 と 5.5、Opus 4.5〜5.5、Sonnet 4.5、4.6、5、5.5、Fable 5 と 5.1 が、リストに入っている(Mythos 系は、別枠)。

### 制約(ドキュメント)
- スキーマで使えない機能がある(数値の最小・最大・倍数、文字列の最小長・最大長など)。使うと 400 エラー。
- オブジェクトには、`additionalProperties: false` が必要。
- 応答が拒否されると、`stop_reason` が `"refusal"` になり、出力が、スキーマに合わないことがある。
- `stop_reason` が `"max_tokens"` のときも、出力が不完全で、スキーマに合わないことがある。
- 文字列の列挙値(enum)の、大文字小文字は、保証されない。
- 引用(citations)を有効にすると、400 エラー。
- 同時に使える、厳密なツールの数や、任意のパラメータの数に、上限がある。
- ドキュメントの機能の互換性の表で、「メッセージのプリフィル」は、JSON 出力と、併用できない、とされている。プリフィルの代わりは、このページには、書かれていない。

### プリフィルが使えなくなったことの根拠(公式)
- Prompting best practices(出典: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices、「Migrating away from prefilled responses」): Claude 4.6 以降のモデルは、最後のアシスタントのターンのプリフィルを、サポートしない。そのリクエストは、400 エラーになる。それ以前のモデルは、引き続き対応している。会話の途中に、アシスタントのメッセージを置くことは、影響を受けない。
- Claude Sonnet 5.5 migration guide(出典: https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide): Sonnet 5.5 は、最後のアシスタントのターンのプリフィルを、400 エラーで拒否する。エラーのメッセージは、「This model does not support assistant message prefill. The conversation must end with a user message.」。Sonnet 4.5、Haiku 4.5 と、それ以前は、受け付ける。出力形式の制御には、構造化出力を使う、と案内されている。
- 同じ移行ガイドに、サンプリングのパラメータ(`temperature` など)の、既定以外の値も、400 エラー、と書かれている。L06 の件と、同じ流れ。

### ページの方法と現行の方法の比較(私の整理)
| | 教材の方法(プリフィル+ストップシーケンス) | 現行の方法(構造化出力) |
|---|---|---|
| 仕組み | Claude に、続きを書かせて、記号で止める | スキーマで、出力の形を制限する |
| 現行モデル | Claude 4.6 以降は、400 エラー | 対応している |
| 形の保証 | なし(Claude の出力任せ) | スキーマに合った、有効な JSON(ただし、`refusal` と `max_tokens` を除く) |
| 後処理 | `strip()` などが必要なことがある | `json.loads()` だけで足りることが多い |
| `stop_reason` | 停止シーケンスで止まれば `stop_sequence`(意図どおり) | `end_turn` であることを確認する |

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| structured data | 構造化データ |
| prefill | プリフィル(アシスタントの返答の、書き出しを、あらかじめ与える) |
| stop sequence | ストップシーケンス(停止シーケンス) |
| structured outputs | 構造化出力 |
| JSON schema | JSON スキーマ |
| strict tool use | 厳密なツール使用 |
| required | 必須(キー) |
| additionalProperties | 追加のプロパティ(スキーマに書いていないキーを許すか) |

## 質疑(学習中の確認)
範囲: このレッスン(L08)と、構造化出力の演習(ex06)。

### 質問1: `required` に、なぜ、この2つを入れるのか(YUTAから)
- YUTAの質問: `"required": ["source", "detail-type"]` を、なぜ必須にするのかがわからない。
- 回答の要点:
  - JSON スキーマでは、`properties` のキーは、既定では、任意(あってもなくてもよい)。`required` に入れたキーは、必ず出力に含まれる。
  - この2つを必須にしたのは、演習の目標(両方が入った JSON が欲しい)に合わせた、私の設計上の選択。EventBridge の決まりではない(EventBridge としての必須かどうかは、確認していない。確かでない)。
  - 必須にすると、後続のコードで、キーを取り出すとき、`KeyError` の危険がなくなる。
  - 私の説明が足りず、理由を書かずに、写させる形になっていた。

### 質問2: スキーマは、自分の望む形にしてよいのか(YUTAから)
- 回答の要点: キー、型、必須かどうかは、アプリが受け取りたい形に合わせて、自分で決める。決まりがある部分(公式の制約)は、`additionalProperties: false` が必要なこと、使えないスキーマの機能があること。

### 質問3: `additionalProperties` を `False` にする意味は
- 質問: `True` にすると何が許され、`False` にすると、何が許されなくなるか。
- YUTAの回答: 未回答(演習を先に進めた)。
- 補足(私の説明): `False` は、スキーマに書いていないキーを、出力に含めることを、許さない指定。構造化出力では、オブジェクトに、`False` が必要。

### 質問4: `output_config` の `format` の書き方(エラーの読み解き)
- 状況: 最初に、`output_config={"format": schema}` と書いて実行した。
- エラー: `BadRequestError: Error code: 400 ... 'message': "output_config.format.type: Input should be 'json_schema'"`
- 読み方: `output_config.format.type` は、「`output_config` の中の `format` の中の `type`」という場所の書き方。「入力は `'json_schema'` であるべき」と言っている。API 側(サーバー側)が拒否した、400 の `BadRequestError`(L05 の `system=None` と、同じ種類)。
- YUTAの回答: 「`format` の `type` が指定されていなかった。`format` の `type` を `json_schema` で指定して、それに続けて `schema` を書くべきであった」
- 評価: ほぼ正解。正確には、「指定されていなかった」のではなく、`schema` の中身が、そのまま `format` の中身になり、`schema` の先頭の `"type": "object"` が、`format` の `type` として読まれた。
- 2つの `type` の違い: `format` の `type` は、形式の種類(`"json_schema"`)。`schema` の中の `type` は、JSON のデータ型(`"object"` など)。

### 質問5: ページの方法と、構造化出力の保証の違い。なぜ `stop_reason` を見たか
- YUTAの回答: 「ページの方法では、構造の保証がされず、Claude の出力に依存する。今回の方法では、`json_schema` の形を指定しているので、有効な `json_schema` であることが保証される。`stop_reason` は、`stop_sequence` で止まったのではないことを確認するため」
- 評価:
  - 前半は、ほぼ正解。ただし、保証されるのは、「有効な `json_schema`」ではなく、「出力が、あなたのスキーマに合った、有効な JSON であること」。スキーマ自体は、あなたが書いたもの。
  - 後半は、不正解。今回は、`stop_sequences` を指定していないので、`stop_sequence` で止まることはない。見る理由は、`refusal`(拒否)と `max_tokens`(上限で切れた)のとき、出力が、スキーマに合わないことがあるため。`end_turn` であれば、保証が成り立つ。
  - 混ざりやすい点: ページの方法では、停止シーケンスで止めるのが意図した動作で、`stop_reason` は `stop_sequence` になる(正常な止まり方)。

## 実機での確認(YUTAの環境、2026-10-21)
- ノートブック: `exercises\ex06_Structured-data\structured-data.ipynb`(ex03 のセットアップとヘルパーをコピーし、TODO とヒントを入れた課題)
- 課題: EventBridge ルールの JSON(`source` と `detail-type`)を、構造化出力で生成する。
- 結果:
  - `stop_reason` は `end_turn`。
  - 取り出した辞書: `{'source': ['aws.ec2'], 'detail-type': ['EC2 Instance State-change Notification']}`
  - 説明文や、コードブロックの記号は、付かなかった。`json.loads(text)` だけで、辞書にできた(`strip()` は不要だった)。
- 取り出しは、`for` と `if` で、`type` が `"text"` のブロックを探す関数にした(`content[0]` の決め打ちを避けた)。
- 気づき:
  - `claude-haiku-5-5` で、構造化出力(`output_config.format`)は、動いた。
  - 教材の方法(プリフィル)を、`claude-haiku-5-5` で動かして、400 エラーになるかは、確認していない(未確認)。
  - 実際のアプリでは、`stop_reason` が `refusal` などで、`text` が `None` になることがあり得る。`json.loads(None)` は、エラーになる。

## 【補足】試験ガイドとの関係
- 試験ガイドに、`temperature` の語はなく、ストリーミングは範囲外とされている。構造化出力や、ツールの JSON スキーマは、私の整理では、ガイドの範囲に近い(対応づけは、私の判断で、公式の指定ではない)。
- このレッスンで、押さえたいこと: 「なぜ、説明文が付くのか」「どう、出力形式を制御するか」と、教材の方法が、現行モデルで使えない理由、現行の方法(構造化出力)との違い。
