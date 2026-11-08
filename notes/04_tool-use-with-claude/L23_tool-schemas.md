# L23 ツールスキーマ(Tool Schemas)

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション4「Claudeでのツール使用」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/tool-schemas (Claude Academy 日本語版、レッスン23/67、7分)
- 読んだ日: 2026-11-08
- 前のレッスン: 「ツール関数」(L22)/ 次のレッスン: 「メッセージブロックの処理」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。
- 注意: 本文テキストのみを要約した。ページの図(スキーマ生成のプロンプト例など)は含まない。
- 注意(演習): このレッスン単独の演習ノートブックは作成しない。L25 までの内容を合わせた1つの演習を、L25 の後に作る方針(YUTAが選択、2026-11-08)。

## 1. このレッスンの目的
ツール関数を Claude に説明する JSON スキーマ(ツール仕様)の書き方と、効果的な description の書き方を理解する。

## 2. スキーマの役割(ページ)
- ツール関数を書いた後の次のステップは、Claude に、関数が期待する引数とその使い方を伝える JSON スキーマを作ること。
- このスキーマは、Claude が**ツールをいつ、どのように呼び出すべきか**を理解するために読むドキュメントとして機能する。

## 3. JSON Schema とは(ページ)
- AI やツール呼び出しに特化したものではなく、長年使われている、広く普及したデータ検証の仕様。
- AI コミュニティが採用したのは、関数のパラメータを記述し、データを検証するのに便利なため。

## 4. ツール仕様の3つの部分(ページ)
| 項目 | 内容 |
|---|---|
| `name` | ツールの明確で説明的な名前(「get_weather」のような) |
| `description` | ツールが何をするか、いつ使うべきか、何を返すか |
| `input_schema` | 関数の引数を記述する、実際の JSON スキーマ |

## 5. 効果的な説明を書く(ページ)
ツールの説明は、Claude が関数をいつ使うべきかを理解するための重要な要素。ベストプラクティス:
1. ツールが何をするかを説明する **3〜4文**を目指す。
2. Claude がいつそれを使うべきかを説明する。
3. どの種類のデータを返すかを説明する。
4. 各引数について詳細な説明を提供する。

## 6. スキーマを生成する簡単な方法(ページ)
ゼロから書かず、Claude 自体を使って生成する。
1. ツール関数のコードをコピーする。
2. Claude に、ツール呼び出し用の JSON スキーマを書くよう依頼する。
3. ツール使用に関する Anthropic のドキュメントを、コンテキストとして含める。
4. Claude に、ベストプラクティスに従った、適切な形式のスキーマを生成させる。

プロンプトの例(ページの英語の趣旨): 「この関数のツール呼び出しを目的とした有効な JSON スキーマ仕様を書いてください。添付のドキュメントに記載されているベストプラクティスに従ってください。」

## 7. コードへの実装(ページ)
```python
def get_current_datetime(date_format="%Y-%m-%d %H:%M:%S"):
    if not date_format:
        raise ValueError("date_format cannot be empty")
    return datetime.now().strftime(date_format)

get_current_datetime_schema = {
    "name": "get_current_datetime",
    "description": "Returns the current date and time formatted according to the specified format",
    "input_schema": {
        "type": "object",
        "properties": {
            "date_format": {
                "type": "string",
                "description": "A string specifying the format of the returned datetime. Uses Python's strftime format codes.",
                "default": "%Y-%m-%d %H:%M:%S"
            }
        },
        "required": []
    }
}
```
- 命名パターン: `関数名` の後に `関数名_schema` を続ける。スキーマを整理でき、対応する関数と簡単に対応付けられる。
- `required` が空 `[]` なのは、`date_format` が省略可能なため(関数側に既定値がある。私の読み)。

## 8. 型安全性の追加(ページ)
```python
from anthropic.types import ToolParam

get_current_datetime_schema = ToolParam({
    "name": "get_current_datetime",
    "description": "Returns the current date and time formatted according to the specified format",
    # ...残りのスキーマ
})
```
- 機能上は厳密には必要ないが、Claude の API でスキーマを使うときの型エラーを防ぎ、コードをより堅牢にする。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| JSON Schema | JSON スキーマ(JSON データの形を記述する仕様) |
| tool schema / tool specification | ツールスキーマ / ツール仕様 |
| `name` / `description` / `input_schema` | ツールの名前 / 説明 / 入力スキーマ |
| required | 必須(キーの一覧) |
| default | 既定値 |
| type safety | 型安全性 |
| `ToolParam` | ツール定義用の型(`anthropic.types` から取り込む) |

## 質疑(学習中の確認)
範囲: このレッスン(L23)の内容。

### 質問1: 天気ツールの description として最も適切なものはどれか
- A: 「天気」
- B: 「指定した都市の現在の天気を取得する。利用者が現在の天気や気温を尋ねたときに使う。気温(℃)、天気の概要、湿度を含む辞書を返す。」
- C: 「天気に関するあらゆる質問に答えるための、非常に便利なツール」
- D: 「get_weather 関数を呼び出す」
- YUTAの回答: B(「どのような結果を得られるかについて詳しく書いてあり、いつ使うのかが明確である」)
- 評価: 正解。理由も、ページの基準(何を返すか、いつ使うか)に沿っている。
  - B が正しい理由: 何をするか(都市の現在の天気を取得)、いつ使うか(現在の天気や気温を尋ねたとき)、何を返すか(気温・天気の概要・湿度を含む辞書)の3点が入っている。
  - 補足: 「引数の説明」もページの基準の一つ。この問題では、引数の説明は `input_schema` 側に書く部分なので、選択肢には出てこない。
  - A: 1語だけで、使う場面も返すものも分からない。
  - C: 「あらゆる質問に答える」は、実際の機能より広く読める。天気以外の質問でも呼ばれる可能性があり、場面も内容も具体性に欠ける。
  - D: 関数名の繰り返しで、新しい情報がない。何を返すか、どの場面で使うかが分からない。
- 補足: `description` は、`name` や `input_schema` と並ぶ、ツール選択の手がかり。曖昧だと、呼ぶべき場面で呼ばれない、または別の場面で呼ばれるおそれがある(私の整理)。ページの「3〜4文」は目安で、必要な3点が入っていれば、短くても足りる。

## 【補足】ページ外の内容(私の整理)
- **L08 とのつながり**: 構造化出力のスキーマと同じ JSON Schema の書き方。ただし、L08 は `output_config` の `format.type="json_schema"` の3段構造だったが、ツールは `input_schema` に直接スキーマを入れる(入れ場所が違う)。構造化出力で出た制約(数値の `minimum`/`maximum` が使えない等)が、ツールのスキーマにも当てはまるかは、確認していない(確かでない)。
- `description` は、設計書で言えば、関数の仕様書にあたる。Claude はこれを頼りに、使うかどうかを判断する。
- 初出の補足: `ToolParam` は、辞書をツール定義として扱うための型の印。実行時の動作は変わらず、型チェックのためのもの(ページの記述に基づく)。
- 試験ガイドとの関係: ガイドには、ツール使用と JSON スキーマによる構造化出力の保証(Task Statement 4.3 の付近)や、`tool_choice` の指定が出てくる。ツールの `description` の書き方がツール選択に影響するという趣旨の記述が、ガイド内にあるかは、まだ確認していない。
