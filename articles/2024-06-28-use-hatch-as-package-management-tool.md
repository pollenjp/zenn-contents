---
title: "パッケージ管理ツールとして hatch を使ってみた"
emoji: "" # 卵
type: "tech"
topics: []
published: false
---

<!--
はてブで投稿: https://pollenjp.hatenablog.jp/entry/2024/07/01/094154
-->

<!--
- hatch
  - build system
  - package management
- 環境を複数定義できる
-->

## きっかけ

Python のパッケージ管理を行うツールは多くありますが、自分はそのお手軽さから rye をよく使っています。

しかしながら、rye は最近のツール故にで EOL なバージョンの Python に対応していないことが多いです。

今回 EOL な Python を使う必要が生じた (※1) ため、Hatch を使ってパッケージ管理してみることにしました (※2)。

使ってみて気になった点の感想をぬるっと書いてみます。

※1 ... そもそも EOL なバージョンを使うべきではないというのはその通りですが、バージョンアップ対応が遅れていたため、一時的な手段として古いものを継続して使っていました。

※2 ... ツールが EOL なバージョンへのサポートを切っていくは普通のことで hatch も例外ではありません。しかし、hatch 自体のバージョンをまだ EOL ではない時の状態に下げれば昔のものも利用することができます。 rye の場合は最近のアップデートが激しいため古いバージョンにすると使用感が大きく崩れます。

## Hatch とは

[Hatch](https://pypi.org/project/hatch/) には主に２つの機能が存在します。１つは build-system としての機能、もう１つはパッケージ管理としての機能です。

Python でパッケージを作成する際は setuptools を使うことが一般的ですが、[PEP 517](https://peps.python.org/pep-0517/) 等に準拠した他の build-system も指定が可能です。 build-system としての hatch もその一つで、簡単にパッケージお設定を記述できるようになっています。 rye もデフォルトでは hatch を build-system として指定しています。

パッケージ管理の機能には様々なものがありますが代表的なものとして以下のようなものが上げられるかと思います。

- パッケージのビルドとアップロード
  - [PEP 518](https://peps.python.org/pep-0518/) の情報を元に wheel ファイルを作成し、 PyPI にアップロードする
    - Python バージョンの指定
    - 依存パッケージの指定
    - build-system の wrap
- 環境の設定と構築 (オプショナル)
  - 本番用・開発用環境の構築や切り替え
  - test/lint/format の実行

Python は単体のファイルでも動作するので、パッケージ管理を行わなくても動作させることは可能ですが、パッケージ管理を行うことで環境の再現性を高めることができます。

## 気になった点

### 依存関係の記述

`pyproject.toml` の `dependencies` に依存関係を **直接** 記述します。 `rye add` や `poetry add` のようなコマンドは存在しないため、手動で記述する必要があります。

詳しい依存関係の整合性をチェックしたり、別ファイルに残したりはしないのでそこだけ注意が必要です。

```toml
dependencies = [ "requests", "click" ] # 例
```

`hatch build`, `hatch publish` などのコマンドを実行することで wheel ファイルを作成し、 PyPI にアップロードすることはできます。

### 複数の仮想環境の設定

Hatch では `pyproject.toml` 内に複数の環境を定義することができます (poetry における group に近いです)。

パッケージ用環境と dev 環境以外に、必要最低限のパッケージだけを入れてテストしたい環境がある場合は便利そうです。

```toml
#
# 開発環境用 (default)
#
[tool.hatch.envs.default]
dependencies = [ "mypy", "ruff", "pyright" ]
[tool.hatch.envs.default.scripts]
typing = [
  "pyright {args:src/pkg_name}",
  "mypy --install-types --non-interactive {args:src/pkg_name}",
]
style = ["ruff check {args:.}"]
fmt = [
  "ruff check --fix {args:.}", # https://docs.astral.sh/ruff/linter/
  "ruff format {args:.}",      # https://docs.astral.sh/ruff/formatter/
  "style",
]
lint-all = ["style", "typing"]
[[tool.hatch.envs.default.matrix]]
python = ["3.12"]

#
# test 用
#
[tool.hatch.envs.all]
dependencies = [ "pytest", "pytest-mock" ]
[tool.hatch.envs.all.scripts]
test = "pytest {args:tests}"
test-cov = "coverage run -m pytest {args:tests}"
cov-report = ["- coverage combine", "coverage report"]
cov = ["test-cov", "cov-report"]
[[tool.hatch.envs.all.matrix]]
python = ["3.10", "3.11", "3.12"]
```

```sh
# default 環境の fmt コマンドを実行したい時
hatch run fmt

# all 環境の test コマンドを実行したい時
hatch run all:test
```

## rye, hatch, poetry の比較

|                       | rye            | hatch | poetry          |
| --------------------- | -------------- | ----- | --------------- |
| Python のインストール | o              | o     | x               |
| build-system の提供   | x              | o     | o (poetry-core) |
| パッケージ作成        | o              | o     | o               |
| 開発環境の構築        | o (2 環境のみ) | o     | o               |
| パッケージ依存性解決  | o              | x     | o               |

## 終わりに

hatch をパッケージ管理ツールとして触ってみた感想を雑に書いてみました。

今まで build-system としての hatch は使ったことがありましたが、パッケージ管理ツールとしては初めて使いました。

既に使ったことのある rye や poetry との違いを感じることができましたが、シングルバイナリという導入のしやすさを考えると個人的には rye がイチオシです。

一方で poetry に近いような細かい設定もできつつ、Python バイナリのインストールも行える hatch は他のツールと比べても一長一短があると感じました。

※ ディレクトリに閉じた環境で Python バイナリを構築したい場合は[こちらの記事](https://zenn.dev/pollenjp/articles/2024-06-27-install-python-at-directory)をご参照ください。
