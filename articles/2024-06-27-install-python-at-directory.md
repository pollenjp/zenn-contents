---
title: "ディレクトリ内に閉じた任意バージョンの Python 環境構築 (python-build-standalone)"
emoji: "🐍"
type: "tech"
topics: ["Python"]
published: true
---

## はじめに

サーバーでちょっとしたスクリプトを実行したいときに使う言語にはいくつか候補がありますが、今回は Python を使うことを考えます。

例えば、共用で使うことが前提のサーバーなどでは、自分が動かすスクリプト以外には影響を与えないようにしたいです。

本記事では、python-build-standalone のバイナリを使い、お手軽にディレクトリ内に閉じた Python 環境構築する方法を紹介します。

※ python-build-standalone バイナリの [Runtime Requirements はこちら](https://gregoryszorc.com/docs/python-build-standalone/main/running.html#runtime-requirements)

## 結論

`./install-python.sh`

```bash
#!/usr/bin/env bash
set -eu -o pipefail

# インストールしたいバイナリを選択 (対応する release date も適宜変更)
# https://github.com/indygreg/python-build-standalone/releases
# 例
# https://github.com/indygreg/python-build-standalone/releases/download/20240415/cpython-3.12.3+20240415-x86_64-unknown-linux-gnu-pgo+lto-full.tar.zst
readonly python_version="3.12.3"
readonly python_build_standalone_release_name="20240415"
readonly python_build_standalone_filename="cpython-"${python_version}"+"${python_build_standalone_release_name}"-x86_64-unknown-linux-gnu-pgo+lto-full.tar.zst"

script_dir=$(
  cd -- "$(dirname "$0")" &>/dev/null
  pwd -P
)
readonly script_dir

readonly local_dir="$script_dir"/.local
readonly tmp_dir="$local_dir/.cache"

readonly python_build_standalone_dir="${local_dir}"/py/"${python_version:?}"
readonly python_installed_dir="${python_build_standalone_dir:?}"/install

if [ ! -d "${python_installed_dir:?}" ]; then
  readonly python_archive="${tmp_dir}/python-${python_version}.tar.zst"
  if [ ! -f "${python_archive}" ]; then
    echo "Downloading Python..."
    mkdir -p "$(dirname "$python_archive")"
    # standalone-python からダウンロード
    curl -L -C - -o "${python_archive}" https://github.com/indygreg/python-build-standalone/releases/download/"${python_build_standalone_release_name}"/"${python_build_standalone_filename}"
    curl -L -C - -o "${python_archive}".sha256 https://github.com/indygreg/python-build-standalone/releases/download/"${python_build_standalone_release_name}"/"${python_build_standalone_filename}".sha256
    echo "Successfully downloaded."

    # check hash
    diff -s <(cut -d' ' -f1 "${python_archive}".sha256) <(shasum -a 256 "${python_archive}" | cut -d' ' -f1)
    echo "Hash is OK."
  fi

  echo "Extracting Python..."
  mkdir -p "${python_build_standalone_dir}"
  # Need zstd pacakge (apt install zstd)
  zstd -d -c "${python_archive}" \
    | tar --strip-components=1 -axf - -C "${python_build_standalone_dir}"
fi

echo "Python is installed at ${python_build_standalone_dir}/install/bin"
```

Python を実行する際は `${python_build_standalone_dir}/install/bin` を PATH に追加してあげれば実行できます。

以下のような wrapper のスクリプトを用意しておくと便利です。

`./python.sh`

```bash
#!/usr/bin/env bash
set -eu -o pipefail

readonly python_version="3.12.3"

script_dir=$(
  cd -- "$(dirname "$0")" &>/dev/null
  pwd -P
)
readonly script_dir
readonly local_dir="$script_dir"/.local
readonly python_build_standalone_dir="${local_dir}"/py/"${python_version:?}"

# ディレクトリに閉じた Python 環境への PATH を追加
export PATH="${python_build_standalone_dir}/install/bin:${PATH}"

# pip install のインストール先指定
export PYTHONUSERBASE="${local_dir}"
# pip install でインストールした CLI ツールへの PATH 追加
export PATH="${local_dir}/bin:${PATH}"

python "$@"
```

※ `PYTHONUSERBASE` については後述します。

## 想定ディレクトリ構成

```txt
some_directory/
|-- .cache/           ... python-build-standalone からダウンロードしたファイルを保存
|-- .local/           ... `~/.local` (XDG_DATA_HOME) と似た役割を持たせる
|   `-- py/3.X,Y/     ... Python のインストール先 (展開先)
|      |-- ...
|      `-- bin/       ... PATH に追加
|
|-- install-python.sh ... Python のインストールスクリプト
`-- python.sh         ... Python のラッパー (シェルスクリプト)
```

## PYTHONUSERBASE に付いて

一部の CLI ツールは `~/.local` や `~/.cache` 等にインストールされるまたは参照する可能性があります。

今回 Python をディレクトリに閉じた環境にインストールしたように、CLI ツールなどもディレクトリに閉じて使いたい場合は、個別に環境変数を必要とする場合があります。

例として、同じくディレクトリに閉じた Pipx (CLI ツール) 環境を構築する例を以下に示したいと思います。

まず、環境変数を設定しないで以下のように `pip install` した場合、 `~/.local` にインストールされてしまいます。

```bash
./python.sh -m pip install --user pipx
```

これは `pip install --user` がデフォルトでパッケージを `~/.local` 以下にインストールするためです。
インストール先を特定のディレクトリに向けるには環境変数 [`PYTHONUSERBASE`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONUSERBASE) で設定する必要があります。

```bash
export PYTHONUSERBASE="$(realpath .)/.local"
./python.sh -m pip install --user pipx
```

このようにすると `./.local/bin/pipx` で pipx を実行できるようになります。

## 注意: アプリ個別のユーザーディレクトリ設定

先の例では pipx をインストールしてみましたが、 pipx 自体も注意して環境変数を設定しないとユーザーディレクトリ以下を使おうとします。

pipx の場合は以下の 3 つに注意が必要です。

- PIPX_HOME ... 仮想環境を配置する場所
- PIPX_BIN_DIR ... 実行ファイルへの symlink を配置する場所
- PIPX_MAN_DIR ... マニュアルページへの symlink を配置する場所

> https://pipx.pypa.io/latest/docs/
>
> ```txt
> Virtual Environment location is ~/.local/share/pipx/venvs.
> Symlinks to apps are placed in ~/.local/bin.
> Symlinks to manual pages are placed in ~/.local/share/man.
>
> optional environment variables:
>   PIPX_HOME              Overrides default pipx location. Virtual Environments
>                         will be installed to $PIPX_HOME/venvs.
>   PIPX_BIN_DIR           Overrides location of app installations. Apps are
>                         symlinked or copied here.
>   PIPX_MAN_DIR           Overrides location of manual pages installations.
>                         Manual pages are symlinked or copied here.
> ```

設定例は次の後述します。

## 実行例

今までの情報を元に、一連の流れを試してみます。

- 独立した Python をインストール
- インストールした Python の pip を利用して pipx をインストール
- pipx を使って pyjokes (CLI ツール) をインストール
- pyjokes を実行

```bash
./install-python.sh
./python.sh -m pip install --user -U pip
./python.sh -m pip install --user pipx

# pipx を使って pyjokes をインストール
local_dir="$(realpath .)/.local"
export PIPX_HOME="${local_dir}/share/pipx/venvs"
export PIPX_BIN_DIR="${local_dir}/bin"
export PIPX_MAN_DIR="${local_dir}/share/man"
./python.sh -m pipx install pyjokes

# pyjoke コマンド実行
./.local/bin/pyjoke
```

## そのた他の方法

ディレクトリ内に閉じていることをキにしなければ、今回紹介した方法以外にも任意の Python バージョンを利用する方法はいくつかあります。

- Python ビルド
  - 公式の Python ビルド
  - pyenv の利用
- 裏で python-build-standalone を利用しているツール等
  - rye ... パッケージマネジメント以外に CLI ツールのインストールにも対応
  - hatch ... パッケージマネジメントを目的とした際に任意の Python バージョンを利用可能
