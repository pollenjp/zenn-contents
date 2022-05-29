---
title: "Poetry: 最初にこれだけおぼえておけば一応使えるメモ"
emoji: "🐍"
type: "tech"
topics: ["poetry", "python"]
published: true
---

[Poetry](https://python-poetry.org/) を使ったことない人に渡すメモ的なものを雑に書きました.
ターゲットはパッケージ管理や仮想環境目的で使う方で, パッケージ作成まではカバーしていません.

## poetry をインストール

- host の python の pip を使って poetry をinstall.
  
  ```sh
  python3 -m pip install poetry
  ```

## プロジェクト初期化

- プロジェクトディレクトリに移動
- ※interactiveにいろいろ聞いてくるが全部脳死Enterでよい

  ```sh
  poetry init
  ```

## virtualenv設定

- poetry は virtualenv を利用して環境を作ってくれる.
- virtualenv のフォルダをどこに作るかを設定

  ```sh
  poetry config --local virtualenvs.in-project true
  ```

- `virtualenvs.in-project true` で基本的にプロジェクトフォルダ直下の `.venv` に作られる
- --local オプションはプロジェクト単位の設定ということで poetry.toml に追記される. もちろん --local を省略して global な設定としても良い.
- どこに virtualenv フォルダを置くかは自由だがプロジェクト直下に置くのが主流
- ※ 実際に virtualenv のディレクトリが作られるのは `poetry install` や `poetry add <pkg name>` 等をしたとき.

## パッケージ追加

- 後は入れたいパッケージを `poetry add package` で入れていく.
- pyproject.toml にパッケージが記述され, poetry.lock に正確なバージョンや依存関係が記述される (これの計算のためにややinstallに時間が掛かる).

  ```sh
  poetry add numpy
  ```

- ※ 削除は `poetry remove <pkg name>` ( pyproject.toml, poetry.lock からも自動削除される)
- ※ 開発ツールの場合は --dev オプションを付ける等もあるがまぁおいおいわかっていけばよい.

## プログラム実行

- poetry run の次に通常のコマンドを書くことで, Poetry が利用している仮想環境 (今回は `.venv` 以下の Python とそれ経由のコマンド・ライブラリ群のこと) に入って実行される.

  ```sh
  poetry run python main.py
  ```

## VSCode での Python Interpreter 選択

- vscode でプロジェクトを開き, Ctrl + Shift + P key から `Python Select Interpreter` みたいなやつを入力して選択して, `.venv/bin/python` をインタプリタに設定

## 新しい環境で pyproject.toml, poetry.lock が存在しているプロジェクトの環境を構築するとき

- poetry.lock があれば正確な依存関係が, pyproject.toml のみであれば install 時に指定したバージョン (していなければ latest) なパッケージ群を一括インストールしてくれる.

  ```sh
  poetry install
  ```

## その他

- [pyenv](https://github.com/pyenv/pyenv): 様々な Python バージョンを利用するのに手放せないツール.
