# L05 システムプロンプト(System Prompts)

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション1「APIを使ったClaudeへのアクセス」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/system-prompts (Claude Academy 日本語版、レッスン5/67、9分)
- 読んだ日: 2026-10-18
- 前のレッスン: 「マルチターン会話」(L04)/ 次のレッスン: 「Temperature」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「質疑」は、学習中の私からの問いとYUTAの回答。「実機での確認」は、YUTAが自分のノートブックで動かした結果。
- 注意: 本文テキストのみを要約した。ページの図は含まない。

## 1. このレッスンの目的
システムプロンプトを使って、Claude がユーザーの入力にどう応答するかをカスタマイズする。一般的な回答ではなく、トーン、スタイル、アプローチを、特定の用途に合わせて形作る。

## 2. システムプロンプトが重要な理由(ページの例)
- 数学の家庭教師チャットボットを考える。学生が「5x + 2 = 3 をxについて解くには?」と聞いたとき、Claude に、答えをそのまま出させたくない。
- 良い家庭教師の振る舞い:
  - 最初は、完全な解答ではなく、ヒントを与える。
  - 学生を、問題に沿って、段階的に、根気強く導く。
  - 類似の問題の解答例を示す。
- 避けたい振る舞い:
  - すぐに、直接的な答えを与える。
  - 学生に、計算機を使うだけでいい、と伝える。

## 3. システムプロンプトの仕組み
- Claude に、応答方法についてのガイダンスを与える。
- プレーンな文字列として定義し、`create` 関数の呼び出しに渡す(引数名は `system`)。
- 利点:
  - Claude に、応答方法のガイダンスを与えられる。
  - Claude は、指定された役割の人物が応答するように、応答しようとする。
  - Claude を、タスクに集中させられる。

基本の形(ページのコード。内容は日本語に置き換えてもよい)
```python
system_prompt = """
You are a patient math tutor.
Do not directly answer a student's questions.
Guide them to a solution step by step.
"""

client.messages.create(
    model=model,
    messages=messages,
    max_tokens=1000,
    system=system_prompt
)
```

## 4. 違いを見る
| 条件 | Claudeの応答(ページの説明) |
|---|---|
| システムプロンプトなし | すぐに、完全な段階的解答を出す。役には立つが、学生自身に考えさせる形ではない |
| 数学の家庭教師のプロンプトあり | 完全な解答の代わりに、導く質問をする(例: 「xを孤立させるための良い最初のステップは何だと思いますか?」) |

## 5. 柔軟な `chat` 関数
システムプロンプトを、関数の中に固定せず、パラメータとして受け取ると、再利用しやすくなる。

```python
def chat(messages, system=None):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
    }

    if system:
        params["system"] = system

    message = client.messages.create(**params)
    return message.content[0].text
```

- 重要な詳細(ページの説明): Claude の API は `system=None` を受け付けない。そのため、システムプロンプトが渡されたときだけ、`system` パラメータを、条件付きで含める必要がある。
- これで、システムプロンプトの有無にかかわらず、同じ関数を呼べる。
  - なし: `chat(messages)`
  - あり: `chat(messages, system=system)`

## 6. まとめ(ページの結び)
システムプロンプトは、意図した目的に対して、一貫性があって、適切に振る舞う AI アプリケーションを作るために不可欠。一般的な応答を、専門的で、役割に合った対話に変える。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| system prompt | システムプロンプト |
| tone / style / approach | トーン / スタイル / アプローチ |
| role | 役割 |
| parameter | パラメータ(引数) |
| keyword argument | キーワード引数 |
| `**params` | 辞書を、キーワード引数として展開して渡す書き方 |

## 質疑(学習中の確認)
範囲: このレッスン(L05)の内容。

### 質問1: `chat()` は、`system=None` の問題を、どう処理しているか
- 質問: ページは「API は `system=None` を受け付けない」と言う。あなたの `chat()` は、どう処理しているか。`if` なしで、常に `params["system"] = system` と書くと、`chat(messages)` は、どうなるか。
- YUTAの回答: 「`system` が `None` ではない場合のみ、`system` パラメータを追加することで、エラーを回避している。`system` が `None` になるため、エラーとなる」
- 評価: 正解。
  - 補足1: `if system:` は、「`None` ではない」より、広い条件。空の文字列(`""`)も、「ない」扱いで、`system` は足されない。
  - 補足2: 辞書を作って、キーを後から足す形にしているのは、「`system` というキー自体を、渡さない」ため。値が `None` のキーを渡すのとは、別の話。

### 実験1(想定外): 引数名のタイプミス
- 状況: `if` の部分をコメントアウトして実験したとき、引数名を `systems` と書いていた。
- 結果: `TypeError: Messages.create() got an unexpected keyword argument 'systems'. Did you mean 'system'?`
- 読み方: 最後の行が、エラーの内容。「`systems` という引数は、ない。`system` の間違いでは?」と、Python が言っている。API に届く前の、Python 側(SDK の引数チェック)のエラー。
- YUTAの対応: 「`systems` にしてしまっていました」と、自分で原因に気づいて、直した。

### 実験2: `system` に `None` を渡す
- 結果: `BadRequestError: Error code: 400 - {'type': 'error', 'error': {'type': 'invalid_request_error', 'message': 'system: Input should be a valid array'}, ...}`
- 読み方: HTTP 400。API のサーバー側が、リクエストを拒否した(入力が不正)。`request_id` は、問い合わせの際に使える、そのリクエストの識別番号。
- 実験1との違い:

| | 実験1 | 実験2 |
|---|---|---|
| 種類 | `TypeError` | `BadRequestError`(400) |
| 起きた場所 | Python 側。API に届いていない | API のサーバー側 |
| 原因 | 引数の名前が、存在しない | 引数の値が、受け付けられない |

- わかったこと: ページの説明どおり、`system=None` は、エラーになる。ただし、この結果は、`claude-haiku-5-5` で、2026-10-18 に動かしたときのもの。
- 気づき: エラーのメッセージは「valid array」とだけ言う。公式のリファレンスは、「文字列、または配列」としている。この食い違いの理由は、確認できていない(確かでない)。

## 実機での確認(YUTAの環境、2026-10-18)
- ノートブック: `exercises\ex03_System-prompts\system-prompts.ipynb`(ex02 のセットアップとヘルパー関数をコピーして、`chat` を `system` 対応に書き換えた)
- `chat` の取り出しは、L03・L04 で直した、`for` と `if` の形のまま。
- 同じ質問(「5x + 2 = 3 を x について解くとどうなる?」)を、`messages` は同じものにして、システムプロンプトの有無だけ変えて比較した。

| 条件 | 結果 |
|---|---|
| なし | すぐに答え(x = 1/5)と、確認の計算まで出した |
| 家庭教師のプロンプト(日本語) | 答えを言わず、「両辺から2を引くと、どうなるか」と問い返し、次のステップを予告した |

- 気づき: 出力はモデルの出力で、毎回完全に同じとは限らない。

## 【補足】ページ外の内容(私の整理)
- 公式リファレンス(Create a Message、リクエストボディ)の `system` パラメータ。確認日 2026-10-18。出典: https://platform.claude.com/docs/en/api/messages/create
  - 型は `optional string or array of TextBlockParam`(省略可。文字列か、テキストブロックの配列)。
  - `null` を受け付ける、とは書かれていない(同じページの他のフィールドには、`null` が書かれている)。
  - 目的は、目標や役割などの、文脈と指示を、Claude に与えること、と説明されている。
- 公式の Prompting best practices(「Give Claude a role」)は、システムプロンプトで役割を設定すると、Claude の振る舞いとトーンが、用途に合う、と説明している。一文でも効果がある、とも書かれている。出典: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- 会話の途中の `system` ロール(未確認の食い違い): 公式の「Using the Messages API」には、会話の途中に、`"role": "system"` のメッセージを足せる機能が、対応モデル(Haiku 5.5、Sonnet 5.5 など)にある、と書かれている。一方、Create a Message のページには、「Messages API に、`system` ロールは、ない」とも書かれている。どちらが現行か、確定できていない(確かでない)。このコースの範囲では、先頭に、トップレベルの `system` を使う。出典: https://platform.claude.com/docs/en/build-with-claude/working-with-messages
- ページのコードの `model = "claude-sonnet-4-5"` は、非推奨(L03 の補足を参照)。演習では `claude-haiku-5-5` を使用した。
- 試験ガイドとの関係(私の対応づけで、公式の指定ではない): システムプロンプトの設計は、エージェントの動作を制御する設計の土台になる。
