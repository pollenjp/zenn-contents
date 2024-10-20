---
title: "PythonのUVを特定のディレクトリに閉じて使う"
emoji: "🐍"
type: "tech"
topics: ["uv", "python"]
published: false
---

## はじめに

Python スクリプトを実行する際には必ず実行環境の Python のバージョンやパッケージ管理について考慮する必要があります。
もちろん Docker 等を使うという方法もありますが、ちょっとしたスクリプト程度のために Docker を使うのは過剰な気もします。

わざわざマシンの構成ついてごちゃごちゃ考えるよりも、実行スクリプトと同じディレクトリに実行環境をささっと構築してしまったほうが楽なケースがあります。

そのためホストマシンの構成についてごちゃごちゃ考えるよりも

以前、[直接 Python を同一ディレクトリにインストールする方法](https://zenn.dev/pollenjp/articles/2024-06-27-install-python-at-directory)を紹介しました。
しかし、実行環境の判定やその他の機能も提供してくれる [astral-sh/uv](https://github.com/astral-sh/uv) を直接インストールしてしまう方も便利なのでこれもまたスクリプト化しました。

## 使い方

まず、以下のコマンドで uv をインストールし `pycmd` というコマンドを生成します。

<https://github.com/pollenjp/install-uv.sh>

```sh
# デフォルトでは current directory に latest なバージョンをインストールします
#
# INSTALL_UV_TARGET_VERSION を指定しない場合は latest なバージョンを自動取得しますが jq command が必要です
curl https://raw.githubusercontent.com/pollenjp/install-uv.sh/refs/heads/main/install-uv.sh | env bash -eu -o pipefail

# jq command をインストールしたくない人は INSTALL_UV_TARGET_VERSION を指定してください
curl https://raw.githubusercontent.com/pollenjp/install-uv.sh/refs/heads/main/install-uv.sh \
  | env INSTALL_UV_TARGET_VERSION=0.4.24 bash -eu -o pipefail

# インストール先を指定したい場合は INSTALL_UV_BASE_DIR を指定してください
curl https://raw.githubusercontent.com/pollenjp/install-uv.sh/refs/heads/main/install-uv.sh \
  | env INSTALL_UV_BASE_DIR=./tmp bash -eu -o pipefail

# ※ 執筆時点での `install-uv.sh` のバージョンは `v0.0.1` なのでそれに合わせる場合は以下
curl https://raw.githubusercontent.com/pollenjp/install-uv.sh/refs/tags/v0.0.1/install-uv.sh | bash -eu -o pipefail
```

任意のコマンドをこの `pycmd` の引数に渡すことで、ローカルにインストールされた uv を参照してくれます。

```sh
$ ./pycmd -- which uv
/path/to/.uv/bin/uv
$ ./pycmd -- uv --version
uv 0.4.24
$ ./pycmd -- uv python install 3.12
Searching for Python versions matching: Python 3.12
Installed Python 3.12.7 in 2.85s
```

## ちょっとした解説

やっていることは極めて単純で、指定したディレクトリに uv をインストールし、 `pycmd` 上で [XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/latest/) をカスタマイズすることで外部のディレクトリを汚さないようにしています。
XDG Base Directory の環境変数を指定すれば uv は適切に参照先を切り替えてくれるように作られています。

## 最後に

ちょこっとした Python スクリプトを社内で共有する際に意外と便利なのでぜひ使って見てください。
