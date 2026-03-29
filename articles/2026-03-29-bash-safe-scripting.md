---
title: "bash スクリプトで踏みやすい罠と対策"
emoji: "🛡️"
type: "tech"
topics: ["bash", "shellscript", "linux"]
published: false
---

## まえがき

bash スクリプトって、なんとなく書いてもそれっぽく動くんですよね。でもそれが罠で、エラーを握りつぶしていたり、意図しない変数を参照していたりしても気づかないことがある。

目次をチラ見して全部にうなづけなかったら一読の価値があるかも、という雑な記事です。

## `set -euo pipefail` はとりあえず入れよう

bash スクリプトの先頭に書いておくだけで、多くの事故を防げます。

```bash
#!/usr/bin/env bash
set -euo pipefail
```

各オプションの意味：

| オプション | 意味 |
|---|---|
| `-e` | コマンドが非ゼロで終了したらスクリプトを即終了 |
| `-u` | 未定義の変数を参照したらエラーで終了 |
| `-o pipefail` | パイプライン内のどこかのコマンドが失敗したら非ゼロを返す |

### NG 例

```bash
#!/usr/bin/env bash

# set -euo pipefail なし
cat not_exist.txt    # エラーになるが続行
echo "続いちゃう"    # 実行される
```

### OK 例

```bash
#!/usr/bin/env bash
set -euo pipefail

cat not_exist.txt    # エラーで即終了
echo "ここには来ない"
```

`-o pipefail` がないと `false | true` の終了ステータスが `0` になります。パイプの途中でコケても気づけないのは危険なのでセットで。

## `local` とコマンド置換の組み合わせに注意

`set -e` を入れていても拾えない罠があります。関数内で `local` と組み合わせたときです。

```bash
#!/usr/bin/env bash
set -euo pipefail

get_value() {
  local result=$(false)  # false は終了ステータス 1 だが…
  echo "ここに来てしまう"  # 実行される！
  echo "$result"
}

get_value
```

`local` コマンド自体の終了ステータスは常に `0` なので、`$(false)` が失敗しても `-e` がトリガーされません。これは有名な bash の罠です。

対策は代入と `local` 宣言を分けること：

```bash
get_value() {
  local result
  result=$(false)  # これは -e が効く
  echo "$result"
}
```

## shellcheck で修行する

[shellcheck](https://github.com/koalaman/shellcheck) は bash/sh スクリプトの静的解析ツールです。上記のような罠を自動で検出してくれます。

インストールは `apt install shellcheck` や `brew install shellcheck` で。

実行は単純：

```bash
shellcheck your_script.sh
```

さらに厳密に検査したい場合は `--enable=all` を付けます：

```bash
shellcheck --enable=all your_script.sh
```

`.editorconfig` に以下を書いておくと、エディタの shellcheck プラグインが自動で適用してくれます：

```ini
[*.sh]
shell=bash
enable=all
```

CI に組み込むのもおすすめです。

## POSIX mode について

`set -o posix` を使うと bash を POSIX 準拠モードで動かせます。コマンド置換などの挙動をより厳密にする効果があります。

https://w3.pppl.gov/info/bash/Bash_POSIX_Mode.html

ただし、bash 固有の便利機能（配列、`[[` 等）が使えなくなるので、既存スクリプトへの適用はコストが高めです。また shellcheck の一部チェックが効かなくなる懸念もあります。

**結論**: `set -euo pipefail` + shellcheck の組み合わせで十分なケースがほとんどです。POSIX mode はオプションとして知っておく程度でよいでしょう。

## まとめ

- `set -euo pipefail` はとりあえず入れる
- `local result=$(cmd)` は罠。宣言と代入を分ける
- shellcheck を CI に入れて静的解析を習慣化する

bash は書けるけど安全に書けているかは別の話。まずは既存スクリプトに shellcheck をかけてみるのが最初の一歩です。
