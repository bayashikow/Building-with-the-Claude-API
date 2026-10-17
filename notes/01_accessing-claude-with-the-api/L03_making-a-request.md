# L03 リクエストを送信する(Making a Request)

- コース: Claude APIを使った構築(Building with the Claude API)
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/making-a-request (Claude Academy 日本語版、レッスン3/67、15分)
- 読んだ日: 2026-10-17
- 前のレッスン: 「APIキーの取得」(L02。内容がAPIキーの取得のみのため、まとめは作成していない)/ 次のレッスン: 「マルチターン会話」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。
- 注意: 本文テキストのみを要約した。ページの文面をそのまま写したものではない。

## 1. このレッスンの目的
Python で、Claude にプロンプトへ応答させる最初のリクエストを送る。基本的なセットアップと、リクエスト・応答の構造を理解する。

## 2. 環境のセットアップ(ページの手順)
1. Jupyter ノートブックで、依存パッケージを入れる: `%pip install anthropic python-dotenv`
2. ノートブックと同じディレクトリに `.env` を作り、`ANTHROPIC_API_KEY="..."` の形で API キーを保存する。
   - 目的: キーをコードから分離し、誤ってバージョン管理にコミットするのを防ぐ。
   - `.env` は必ず `.gitignore` に追加する。
3. 環境変数を読み込み、クライアントを作る。

```python
from dotenv import load_dotenv
load_dotenv()

from anthropic import Anthropic

client = Anthropic()
model = "claude-sonnet-4-5"
```

| 行 | 役割(補足を含む。下の【補足】参照) |
|---|---|
| `load_dotenv()` | `.env` を探して、中身を環境変数に登録する |
| `client = Anthropic()` | API を呼ぶ窓口を作る。キーを渡していないが、環境変数 `ANTHROPIC_API_KEY` が使われる |
| `model = ...` | 使うモデル名を変数に入れるだけ。API はまだ呼ばれない |

## 3. create 関数
API リクエストの中心は `client.messages.create()`。必要なパラメータは3つ。

| パラメータ | 内容 |
|---|---|
| `model` | 使う Claude モデルの名前 |
| `max_tokens` | 応答の長さに対する安全上の上限。目標値ではない |
| `messages` | Claude に送る会話履歴 |

- `max_tokens` は安全機構。1000 に設定すると、Claude がまだ話すことがあっても 1000 トークンで生成を止める。
- Claude は上限に届こうとはしない。適切だと考える内容を書き、上限に達したら止まるだけ。

## 4. メッセージ
- チャットアプリのように、あなたと Claude の会話を表す。
- 種類は2つ: ユーザーメッセージ(人間が書いたもの)/ アシスタントメッセージ(Claude が生成した応答)。
- 各メッセージは、`role`(`"user"` か `"assistant"`)と `content`(実際のテキスト)を持つ辞書。

## 5. 最初のリクエスト
```python
message = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=[
        {
            "role": "user",
            "content": "What is quantum computing? Answer in one sentence"
        }
    ]
)
```
- 実行すると、Claude がリクエストを処理し、生成されたテキストとリクエストのメタデータを含む応答オブジェクトを返す。

## 6. 応答の取り出し
- 応答オブジェクトには多くの情報があるが、通常必要なのは生成されたテキストだけ。
- 取り出し方: `message.content[0].text`

## 重要ポイント(ページの結び)
基本を理解したら、いろいろなプロンプトを試し、より複雑なやり取りを組み立てていける。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| request | リクエスト |
| model | モデル |
| max_tokens | 最大トークン数 |
| messages | メッセージ(会話履歴) |
| role | ロール(役割) |
| user message | ユーザーメッセージ |
| assistant message | アシスタントメッセージ |
| response object | 応答オブジェクト |

## 質疑(学習中の確認)
範囲: このレッスン(L03)の内容。

### 質疑0: `load_dotenv()` 等の意味(YUTAからの質問)
- 質問: ページのセットアップコードは何をするのか。ファイルを作れという意味か。
- 回答の要点: 上の表のとおり。`.env` とノートブックの2ファイルを使う前提。ページに置き場所の指定はない。

### 質問1: `message.content[0].text` の `[0]` は何か。なぜ `message.text` ではないのか
- YUTAの回答: 「messageの1番目のブロックを取得している。1番目には回答が格納されている。`message.text` と書いた場合、回答以外の情報も含まれるため」
- 評価: 部分的に正解。
  - 合っている点: `[0]` は `content` のリストの1番目のブロック。今回は、そこに回答の文章が入っている。
  - 不正確な点:
    - 「`message.text` だと回答以外も含まれる」は違う。私の理解では `message` に `text` 属性はなく、書いてもエラーになる(SDK での確認はできていない。確かでない)。
    - 「1番目に回答がある」は常にそうとは限らない。`content` はリストで、ブロックの種類が混ざることがある(`text`、`tool_use` など)。
- 正しい構造: `message` は応答全体(停止理由や使用量なども持つ)。本文は `content` というリストの中の、`text` ブロックにある。

### 質問2: `content[0]` が `text` ブロックでなかったら、どう取り出すか
- YUTAの回答(1回目): `message.content[0].text` とする → 質問の答えになっていない(同じコードの繰り返し)。
- ヒントを出して再回答: 「`content` から、`text` を示すブロックを抽出する。`for` や `if` を利用する」
- 評価: 正解。「ブロックの種類を見てから取り出す」という考え方で合っている。
- 参考のコード形:
```python
for block in message.content:
    if block.type == "text":
        print(block.text)
```
- 取り捨て: `content[0].text` は短いが、ツールなどでブロックが混ざると壊れる。`for` と `if` は長いが、構成が変わっても動く。ツールを渡さない単純な呼び出しなら `content[0].text` で足りる。ツールを使い始めた段階で切り替える。

## 実機での確認(YUTAの環境、2026-10-17)
- `.env` はリポジトリ直下、ノートブックは `exercises\ex01_Making-a-request\` にある(2階層下)。
- ノートブック上で `print(load_dotenv())` が `True` を返した。サブフォルダからでも、親のフォルダの `.env` を見つけられる。
- 実行している Python は `.venv\Scripts\python.exe`。
- コードの実行は成功し、日本語の質問(「量子計算とは何ですか?一文で回答してください」の趣旨)に対して、一文の回答が返った。
- 気づき: ノートブックの質問文に誤字(「なんですか」→「はんですか」)があった。動作には影響しない。

## 【補足】ページ外の内容(私の整理)
- ページのコードの `model = "claude-sonnet-4-5"` は、公式ドキュメントのモデル一覧(2026-10-17に確認済み)では、2026-09-30 に非推奨になり、廃止予定が 2026-11-30(暫定)。置き換え先は `claude-sonnet-5-5`。
- 現行のモデルID(同上): `claude-fable-5-1`、`claude-opus-5-5`、`claude-sonnet-5-5`、`claude-haiku-5-5`。ハイフン区切りで、ドット区切りではない(ノートブックでは、安価な `claude-haiku-5-5` を使用)。
- API を呼ぶと、少額の料金がかかる(従量課金)。演習では安いモデルと小さい `max_tokens` を使う。
- `content` のブロックの種類(`text`、`tool_use` など)と `stop_reason` の関係は、L01 の補足と、Platform 101 で学んだ内容につながる。
- 試験ガイドとの関係(私の対応づけで、公式の指定ではない): 1.1(エージェントループと `stop_reason`)の土台になる。
