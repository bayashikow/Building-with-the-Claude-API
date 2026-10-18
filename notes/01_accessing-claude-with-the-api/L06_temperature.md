# L06 Temperature

- コース: Claude APIを使った構築(Building with the Claude API)/ セクション1「APIを使ったClaudeへのアクセス」
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/temperature (Claude Academy 日本語版、レッスン6/67、7分)
- 読んだ日: 2026-10-18
- 前のレッスン: 「システムプロンプト」(L05)/ 次のレッスン: 「レスポンスストリーミング」
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「実機での確認」は、YUTAが自分のノートブックで動かした結果。
- 注意: 本文テキストのみを要約した。ページの図とインタラクティブなデモは含まない。
- 注意: このレッスンのコード(`temperature` を渡す部分)は、YUTAの環境では動かなかった。理由と対応は「実機での確認」と【補足】に記録した。

## 1. このレッスンの目的
Temperature は、Claude の応答が、どの程度予測可能か、あるいは創造的かを、制御するパラメータ。使い方を理解して、AI アプリケーションを改善する。

## 2. Claude がテキストを生成する流れ(3ステップ)
プロンプト(例: 「What do you think?」)を送ると、次の3ステップを経る。
1. トークン化(Tokenization): 入力を、小さなチャンクに分割する。
2. 予測(Prediction): 次に来る可能性のある単語の確率を計算する。
3. サンプリング(Sampling): その確率に基づいて、トークンを選ぶ。

- 例: 「about」に30%、「would」に20%、「of」に10%、などの確率を割り当てる。
- モデルは1つのトークンを選び、このプロセス全体を繰り返して、完全な文を作る。

## 3. Temperature の役割
- 0 から 1 の間の小数値。上の選択確率に、直接影響する。「創造性のダイヤル」にたとえられる。

| 値 | 動き | 出力 |
|---|---|---|
| 低い(0 に近い) | ほぼ常に、最も確率の高いトークンを選ぶ(決定論的) | 予測しやすい |
| 高い(1 に近い) | 確率が、選択肢全体に、より均等に分散する | 多様で、創造的 |

- インタラクティブなデモの説明: temperature 0.0 では、「about」が 100% の確率になり、完全に決定論的。1.0 では、確率がすべての可能なトークンに、より均等に分散し、ランダム性と創造性が生まれる。

## 4. 適切な Temperature の選び方(ページの目安)
| 範囲 | 向いているタスク |
|---|---|
| 低い(0.0〜0.3) | 事実に基づく応答、コーディング支援、データ抽出、コンテンツモデレーション |
| 中程度(0.4〜0.7) | 要約、教育コンテンツ、問題解決、制約付きの創作文章 |
| 高い(0.8〜1.0) | ブレインストーミング、創作文章、マーケティングコンテンツ、ジョーク生成 |

## 5. コードでの実装(ページのコード)
`chat` 関数に `temperature` のサポートを足す。

```python
def chat(messages, system=None, temperature=1.0):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
        "temperature": temperature
    }

    if system:
        params["system"] = system

    message = client.messages.create(**params)
    return message.content[0].text
```

- 主な変更点: パラメータに `temperature=1.0` を足し、`params` の辞書に `"temperature": temperature` を含める。

効果のテスト(ページの例): 映画のアイデアを、異なる設定で生成する。
```python
# 低い温度 - より予測可能
answer = chat(messages, temperature=0.0)

# 高い温度 - より創造的
answer = chat(messages, temperature=1.0)
```
- temperature 0.0 では、似た応答が、一貫して得られる場合がある。1.0 では、テーマ、キャラクター、プロット要素に、はるかに多様性が出る。

## 6. 重要なポイント(ページの結び)
- temperature は、異なる出力を、保証するものではない。それらが得られる確率を変えるだけ。高い temperature でも、似た応答が、出ることがある。
- 使い分け:
  - 一貫した事実に基づく応答 → 低い temperature
  - 創造的なブレインストーミング → temperature を上げる
  - その中間 → 中程度の temperature(ほとんどの一般的なタスクに適する)
- 重要なのは、temperature の選択を、特定のユースケースに合わせること。

## 用語(英語/日本語)
| 英語 | 日本語 |
|---|---|
| temperature | 温度(Temperature) |
| tokenization | トークン化 |
| prediction | 予測 |
| sampling | サンプリング |
| deterministic | 決定論的 |
| probability distribution | 確率分布 |

## 実機での確認(YUTAの環境、2026-10-18)
- ノートブック: `exercises\ex04_Temperature\temperature.ipynb`(ex03 のセットアップとヘルパー関数をコピーして、`chat` に `temperature` を足した)
- 動かしたところ、次のエラーが出た。
  `TypeError: Messages.create() got an unexpected keyword argument 'temperature'`
- 読み方: `systems` のときと同じ種類(Python 側の引数チェック)。「`create()` に、`temperature` という引数がない」と言っている。API には届いていない。
- YUTAの見立て: 「haiku だと temperature をサポートしていないかも?」→ 公式ドキュメントの記述と、方向は合っている(下の【補足】参照)。ただし、観察したエラーは、API 側の 400 ではなく、SDK 側の `TypeError` だった。
- 確認の実験:
  ```python
  import inspect
  print("temperature" in inspect.signature(client.messages.create).parameters)
  ```
  結果は `False`。YUTAの環境の SDK では、`create()` が `temperature` を引数に持たない。(使用中の SDK は `anthropic` 1.15.0。2026-10-18)
- 対応(YUTAの判断): 実験(0.0 と 1.0 の比較)は飛ばして、概念だけ理解して、次のレッスンへ進む。理由: 公式ドキュメントが「非推奨」としていること、試験ガイドに記載がないこと、実験のために、モデルを変える手間が、釣り合わないこと。
- 取らなかった案:
  - `extra_body={"temperature": 0.0}` で、API に直接渡し、`temperature` を設定できる古いモデルで試す(古いモデルの名前と、利用できるかを、調べる必要がある)。
  - `claude-haiku-5-5` のまま、`extra_body` で `temperature=1.0` だけを渡す(動く可能性は高いが、比較にならない)。

## 【補足】ページ外の内容(私の整理)
- 公式リファレンス(Create a Message、リクエストボディの `temperature`)。確認日 2026-10-18。出典: https://platform.claude.com/docs/en/api/messages/create
  - 「Deprecated(非推奨)」と表示されている。
  - "Models released after Claude Opus 4.6 do not support setting temperature."(Opus 4.6 より後のモデルは、temperature を設定できない)
  - 1.0 は、後方互換のために受け付けられ、他の値は、400 エラーで拒否される。
  - 範囲は 0.0〜1.0、既定値は 1.0。0.0 でも、結果は完全には決定論的ではない、とも書かれている。
  - ページには、Haiku 5.5 の名前は出てこない。Haiku 5.5 が、「Opus 4.6 より後」に当たるかは、ページからは確定できない(番号から、可能性は高いが、これは私の推測。確かでない)。
- 今回の `TypeError`(SDK 側)と、ドキュメントの 400 エラー(API 側)は、層が違う。SDK が、非推奨に合わせて、引数の定義から外した可能性がある。この点は、確認できていない(確かでない)。
- 試験ガイドとの関係: ガイドの全文を検索したが、`temperature` という語は、一度も出てこない。試験に出るかは、ガイドからは、わからない。
- このコースのページの、「0.0 と 1.0 で、出力の多様性が変わる」という説明は、概念としては有効。ただし、コードで再現するには、`temperature` を設定できるモデルと、SDK の使い方が必要で、新しい環境では、そのままでは動かない。
