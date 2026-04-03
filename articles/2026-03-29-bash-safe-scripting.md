---
title: "bashスクリプトで踏みやすい罠:pipefail/inherit_errexit/コマンド置換・プロセス置換/shellcheck"
emoji: "🛡️"
type: "tech"
topics: ["bash", "shellscript", "linux"]
published: true
---

## まえがき

目次をチラ見して「あーはいはい」とならなかったら一読の価値があるかも、という雑な記事です。

## `set -euo pipefail` はとりあえず入れよう

bash スクリプトの先頭に書いておくだけで、多くの事故を防げます。

```bash
#!/usr/bin/env bash
set -euo pipefail
```

各オプションの意味：

| オプション    | 意味                                                     |
| ------------- | -------------------------------------------------------- |
| `-e`          | コマンドが非ゼロで終了したらスクリプトを即終了           |
| `-u`          | 未定義の変数を参照したらエラーで終了                     |
| `-o pipefail` | パイプライン内のどこかのコマンドが失敗したら非ゼロを返す |

### NG 例 (set -euo pipefail なし)

```bash
#!/usr/bin/env bash

# set -euo pipefail なし
cat not_exist.txt    # エラーになるが続行
echo "続いちゃう"    # 実行される
```

### OK 例 (set -euo pipefail あり)

```bash
#!/usr/bin/env bash
set -euo pipefail

cat not_exist.txt    # エラーで即終了
echo "ここには来ない"
```

`-o pipefail` がないと `false | true` の終了ステータスが `0` になります。パイプの途中でコケても気づけないのは危険なのでセットで。

## `shopt -s inherit_errexit` もセットで

`set -euo pipefail` を入れても、`$()` コマンド置換で作られるサブシェルにはデフォルトで `errexit` が継承されません。サブシェル内でコマンドが失敗しても `set -e` が効かないケースがあります。

対策は `shopt -s inherit_errexit` をあわせて設定することです：

### NG 例 (コマンド置換で呼ばれたサブシェルでエラーがあっても止まらない)

```sh
#!/usr/bin/env bash
set -euo pipefail
# shopt -s inherit_errexit
# <- comment out

echo "start-main"

hello() {
    echo "start-hello"
    failing_command
    echo "end-hello"
}

x=$(hello)
echo "$x"

echo "end-main"
```

コマンド実行

```sh
./script.sh
```

出力

```txt
start-main
./script.sh: line 9: failing_command: command not found
start-hello
end-hello
end-main
```

### OK 例 (`shopt -s inherit_errexit` を入れるとサブシェル内のエラーも拾える)

```sh
#!/usr/bin/env bash
set -euo pipefail
shopt -s inherit_errexit

echo "start-main"

hello() {
    echo "start-hello"
    failing_command
    echo "end-hello"
}

x=$(hello)
echo "$x"

echo "end-main"
```

以下は出力です。

```txt
start-main
./script.sh: line 9: failing_command: command not found
```

これでサブシェル内でも `errexit` が有効になります。shellcheck の [SC2311](https://www.shellcheck.net/wiki/SC2311) がこの問題を警告してくれます。

## shellcheck で修行する

[shellcheck](https://github.com/koalaman/shellcheck) は bash/sh スクリプトの静的解析ツールです。上記のような罠を自動で検出してくれます。

インストールは `apt install shellcheck` や `brew install shellcheck` で。[mise](https://mise.jdx.dev/) を使っているなら `mise use shellcheck` でも OK です。

実行は単純：

```bash
shellcheck your_script.sh
```

個人的にはもっと厳密にしたほうが良いので `--enable=all` を付けます：

```bash
shellcheck --enable=all your_script.sh
```

`.shellcheckrc` に以下を書いておくと、エディタの shellcheck プラグインやコマンドが自動で適用してくれます：

```.shellcheckrc
enable=all
```

CI に組み込むのもおすすめです。

## 引数内でのコマンド置換に注意

`set -e` を入れていても、引数の中で `$(cmd)` を使うと失敗を拾えないことがあります。

### NG 例 (引数内コマンド置換)

```bash
# NG: failing_command が失敗しても set -e で止まらない
some_command "$(failing_command)"
```

`$(failing_command)` が失敗しても、`some_command` に空文字列が渡って処理が続いてしまいます。

### OK 例 (コマンド置換→変数に受ける)

```bash
# OK: まず変数に受ける（ここで失敗すれば set -e が効く）
result=$(failing_command)
some_command "${result}"
```

一度変数に詰めることで、`set -e` が正しく効くようになります。

### `export` や `local` も注意

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

## プロセス置換も同様

`<(cmd)` を使ったプロセス置換も同じ罠があります。

### NG 例 (プロセス置換)

```bash
# NG: failing_command が失敗しても止まらない
while read -r line; do
  echo "${line}"
done < <(failing_command)
```

### OK 例 (変数に受ける)

```bash
# OK: まず変数に受ける
data=$(failing_command)
while read -r line; do
  echo "${line}"
done <<< "${data}"
```

ヒアストリング (`<<<`) で渡せば、失敗は変数代入の時点で `set -e` が検知します。

### この代替パターンのトレードオフ

`data=$(failing_command)` + `<<<` パターンには以下の制約がある：

1. **メモリ使用量**: コマンドの出力全体をメモリに読み込む → 大量出力ではスケールしない
2. **末尾改行の消失**: コマンド置換 `$()` は末尾の改行を取り除く bash の仕様
   - 例: `printf "foo\n\n"` の出力末尾の改行が消える
   - 末尾改行が意味を持つデータでは、元のプロセス置換と完全に等価にならない

### ストリーム処理しつつ失敗を検知するパターン

ストリームのまま処理したい場合の代替パターン。

#### パターンA: 一時ファイルを使う

```bash
tmpfile=$(mktemp)
failing_command > "${tmpfile}"
exit_status=$?
if [[ ${exit_status} -ne 0 ]]; then
  rm -f "${tmpfile}"
  exit "${exit_status}"
fi
while IFS= read -r line; do
  echo "${line}"
done < "${tmpfile}"
rm -f "${tmpfile}"
```

- ストリーム処理可能（メモリ制約なし）
- `$()` を使わないため末尾改行も保持される
- ただし一時ファイルのクリーンアップが必要（`trap` で確実に行うと安全）

#### パターンB: `pipefail` + 通常のパイプ

`set -o pipefail` が有効な環境なら、通常のパイプで失敗を検知できる：

```bash
# set -o pipefail 環境ではパイプ左辺の失敗を自動検知
failing_command | while IFS= read -r line; do
  echo "${line}"
done
```

- ストリーム処理可能
- `failing_command` が失敗すると `pipefail` によりパイプライン全体が非ゼロで終了し、`set -e` がスクリプトを停止する
- ただし `while` ループはサブシェルで実行されるため、ループ内で設定した変数は親シェルから参照できない
- `set -o pipefail` なしでは左辺の失敗を検知できないため注意

## 【補足】`set -o pipefail` と `head` は相性悪いので `awk` とかで代用しよう

`set -o pipefail` 環境下で `some_command | head -n 5` のようなパイプを使うと、`head` が必要な行を読み終えた時点でパイプを閉じ、左側のコマンドが SIGPIPE を受けて非ゼロで終了します。`pipefail` はパイプ内の最後の非ゼロ終了コードを拾うため、スクリプトが意図せず停止することがあります。

### NG 例 (pipefail + head)

```bash
# NG: pipefail 環境下で SIGPIPE によりスクリプトが停止することがある
some_command | head -n 5
```

### OK 例 (pipefail + awk)

```bash
# OK: awk を使えば SIGPIPE を発生させずに先頭 N 行を取得できる
some_command | awk 'NR<=5'
```

## 【補足】POSIX mode について

`set -o posix` を使うと bash を POSIX 準拠モードで動かせます。コマンド置換などの挙動をより厳密にする効果があります。

ただし、bash 固有の機能の一部に制限が加わるので、既存スクリプトへの適用はコストが高めです。また POSIX mode を有効化した上でのshellcheck の一部チェックが効かなくなる懸念もあります。

制限周りについては [こちらのページ (Bash POSIX Mode)](https://w3.pppl.gov/info/bash/Bash_POSIX_Mode.html) 等にまとまっています。

**結論**: `set -euo pipefail` + shellcheck の組み合わせで十分なケースがほとんどです。POSIX mode はオプションとして知っておく程度でよいでしょう。

## まとめ

- `set -euo pipefail` + `shopt -s inherit_errexit` をスクリプト冒頭に（サブシェルにも `errexit` を継承させる）
- コマンド置換・プロセス置換 (`$(cmd)` / `<(cmd)`) は一度変数に受けてから使う（`local` 宣言との合わせ技も罠なので宣言と代入は分ける）
- `pipefail` + `head` は SIGPIPE の罠。`awk 'NR<=N'` で代替
- shellcheck は `.shellcheckrc` に `enable=all` を書いて CI に組み込む
- POSIX mode はお好みで（`set -euo pipefail` + shellcheck で十分なケースがほとんど）

bash は書けるけど安全に書けているかは別の話。まずは既存スクリプトに shellcheck をかけてみるのが最初の一歩です。
