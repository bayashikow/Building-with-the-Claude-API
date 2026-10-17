# L02 APIキーの取得(Getting an API Key)

- コース: Claude APIを使った構築(Building with the Claude API)
- 出典: https://academy.claude.com/ja/courses/building-with-the-claude-api/getting-an-api-key (Claude Academy 日本語版、レッスン2/67、1分)
- 読んだ日: 2026-10-17
- 前のレッスン: 「APIへのアクセス」(L01)/ 次のレッスン: 「リクエストを送信する」(L03)
- 凡例: 特に印のない記述は、ページ本文の内容。【補足】は、ページ外の内容(私=Claudeの補足)。「実機の環境設定」は、YUTAが自分の PC で実際に行った作業の記録(ページ本文にはない)。
- 注意: 本文テキストのみを要約した。ページの図(画面キャプチャ)の内容は含まない。
- 注意: API キーそのものは、この文書にも、チャットにも、書かない。

## 1. このレッスンの目的
次のレッスンで Anthropic API にリクエストを送るために、シークレット API キーを作る。キーの作り方を説明する。

## 2. 手順(5ステップ)
1. ブラウザで https://platform.claude.com/ を開き、Anthropic アカウントでログインする。
2. メインダッシュボードの右上付近にある「Get API Keys」ボタンを押す。
3. ページ右上の「Create Key」ボタンを押す。
4. ワークスペース「Default」を選び、キーの名前を入力する。名前は、生成したキーを見分けるためのもの(ページの例は「Anthropic Course」)。
5. ポップアップに表示された API キーをコピーして保管する。
   - キーは一度しか表示されない。
   - 閉じてしまったら、古いキーを削除して、もう一度生成する。

## 【補足】ページ外の内容(私の整理)
- L01 のとおり、API キーはクライアント(ブラウザ等)に置かず、サーバー側で安全に保管する。L03 では、`.env` に保管し、`.gitignore` で Git の対象外にする。
- API の利用は従量課金。YUTA は、料金がかかる点を了承して進めた。新規ユーザーに少額の無料クレジットがあるとの情報があるが、金額は確認できていない(確かでない)。
- 演習では、安いモデル(`claude-haiku-5-5`)と小さい `max_tokens` を使い、費用を抑える。

## 実機の環境設定(YUTAの環境、2026-10-17)
ページ本文にはない作業の記録。コースは「Jupyter ノートブックで `%pip install ...`」と書くのみで、仮想環境の作成や Jupyter の導入は説明していない。以下は、YUTAの環境(Windows、PowerShell、VS Code)で、私が案内して、YUTAが実行した手順。

### 環境の確認結果
| 確認 | 結果 |
|---|---|
| `py --version` | 認識されない(Windows の Python ランチャーが無い) |
| `python --version` | Python 3.13.5。実体は Anaconda(`where python` で、Anaconda と WindowsApps が出た) |
| `uv` | 入っていなかった |

### 作業の流れ
| 手順 | 内容 | 理由 |
|---|---|---|
| 1. 作業フォルダ | `C:\WORK\10_study\100_anthropic_exam\20_Building-with-the-Claude-API\`(Git リポジトリ。公開設定) | コースの演習用 |
| 2. `.gitignore` | `.env`、`.venv/`、`__pycache__/`、`.ipynb_checkpoints/` を対象外にした | キーと環境を、公開リポジトリに上げないため |
| 3. 仮想環境の作成 | `python -m venv .venv`(リポジトリ直下) | コースのパッケージを、Anaconda 全体から分離するため。`uv` ではなく標準の `python -m venv` を選んだ(`uv` は未導入) |
| 4. 仮想環境の有効化 | PowerShell で `Activate.ps1` を実行。スクリプトの実行がブロックされたため、`Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned` を実行してから、`.venv\Scripts\Activate.ps1` を実行した。プロンプトに `(.venv)` が出た | `-Scope Process` は、そのターミナルを閉じるまでの間だけ許可する設定。PC 全体の設定は変えていない |
| 5. パッケージの導入 | `pip install anthropic python-dotenv` | コースの `%pip install` と同じ内容を、事前に行った |
| 6. ノートブック実行環境 | `pip install ipykernel` | VS Code でノートブックを動かすために必要。コースのページには無い。私が案内した |
| 7. API キーの保管 | リポジトリ直下に `.env` を作り、`ANTHROPIC_API_KEY=...` を保存した。キーはチャットに貼っていない。`git status` で `.env` が出ないこと(対象外)を確認した | キーを Git に上げないため |
| 8. 読み込みの確認 | ノートブックで `print(load_dotenv())` が `True`。`sys.executable` が `.venv\Scripts\python.exe` | 2階層下のサブフォルダのノートブックからでも、親フォルダの `.env` が見つかった |
| 9. ノートブックの場所 | `exercises\ex01_Making-a-request\making-a-request.ipynb` | L03 の演習用 |

### 導入したパッケージ(`pip list` より)
- `anthropic` 1.15.0
- `python-dotenv` 1.2.4
- `ipykernel` 7.4.0(と、その依存の `jupyter_client` など)
- `notebook`(ブラウザ版の Jupyter)は入れていない。VS Code で開くため。

### つまずいた点と対応
| 症状 | 対応 |
|---|---|
| VS Code に、`uv` と Python を入れるかを尋ねるダイアログが出た | 「キャンセル」を案内した(仮想環境は `python -m venv` で作るため) |
| 「無効な Python インタープリター」のエラー | 仮想環境の Python 自体は動いた。VS Code の再読み込み(Reload Window)と、インタープリターの再選択を案内した。最終的に、ログで `.venv` が選ばれた記録を確認した |
| ログに「このリソースには Python 環境が設定されていません」の警告 | Python Environments 拡張機能が環境を認識できていないことを示す。ノートブックはカーネルを別に選ぶため、実行はできた |

### 補足: 仮想環境の有効化(activate)とは(私の説明。ページ外)
- 仮想環境の作成(`python -m venv .venv`)は、`.venv` フォルダに、専用の Python とパッケージの置き場を作るだけ。作っただけでは、ターミナルはまだ Anaconda などの Python を使っている。
- 有効化(`.venv\Scripts\Activate.ps1`)は、そのターミナルの中だけで、`python` と `pip` が `.venv` のものを指すように、切り替える操作。成功すると、プロンプトの先頭に `(.venv)` が付く。
- 有効化したあとの `pip install` は、`.venv` の中にだけ入る。Anaconda 側は汚れない。`pip list` で、入っているパッケージを確認できる(今回の結果は上のとおり)。
- 有効化は、ターミナルごとの設定。新しいターミナルを開くたびに、もう一度必要になる(VS Code の設定によっては自動で行われることがある。今回の環境では未確認)。
- 有効化の確認方法: プロンプトの `(.venv)` を見る。または `where python`(PowerShell では `Get-Command python`)で、先頭が `.venv\Scripts\python.exe` かを見る。
- 有効化を解く(元に戻す)のは `deactivate`。ターミナルを閉じても戻る。
- 実行ポリシーの対応との関係: PowerShell は、既定でスクリプトの実行を制限していることがあり、`Activate.ps1` がそれに当たってブロックされた。`Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned` は、そのターミナルの間だけ許可を緩める。新しいターミナルでは、有効化のたびに、同じ対応が必要になる可能性がある。
- 有効化しなくても使える方法: `.venv\Scripts\python.exe -m pip install ...` のように、`.venv` の Python を直接指定する。今回は使っていない。
- ノートブックとの関係: VS Code のノートブックは、ターミナルの有効化とは無関係に、右上の「カーネルの選択」で選んだ Python で動く。今回は `.venv` を選び、`sys.executable` で確認した。ターミナルの有効化と、カーネルの選択は、別々の設定。

### 補足の判断(取り捨て)
- Anaconda の Python をそのまま使えば、手順は減るが、他のプロジェクトのパッケージと混ざる。仮想環境にして、依存を分けた。
- ブラウザ版の Jupyter(`notebook`)ではなく、普段使いの VS Code で動かす方を選んだ。
- コースの `%pip install anthropic python-dotenv` は、すでに入れてあるため、ノートブックでは省略できる。
