---
title: "そのSetup系GitHub Actions、ちゃんとキャッシュできていますか？ (@actions/tool-cache)"
emoji: "🚢"
type: "tech"
topics: ["github-actions", "tool-cache"]
published: false
---

<!--
TODO: planning to published at Qiita
-->

<!--

構成

- TL;DR
- Setup 系 Action の定義
  - この記事で言及する Setup 系 Action の定義
- キャッシュが利用されていない例
- キャッシュが利用されている例
- Tool Cache

-->

## はじめに

こんにちは、ぽれん(@'ω'@)です。

私は最近、 shfmt や shellcheck を Self-hosted Runner で使いたかったのですが、どうにもちゃんとキャッシュしてくれる 3rd Party Action が見つからなかったため、自分で作ってみました。

- [pollenjp/setup-shfmt](https://github.com/pollenjp/setup-shfmt)
- [pollenjp/setup-shellcheck](https://github.com/pollenjp/setup-shellcheck)

今まで workflow を書くときは `run` step の中で shell をこねくり回すことが多く、ちょうど嫌気が指していたタイミングだったため TypeScript の Custom Action を学ぶ良い機会となりました。

また別の話ですが、私は普段 [astral-sh/uv](https://github.com/astral-sh/uv) というツールを好んで使っているのですが、公式で提供されている [astral-sh/setup-uv](https://github.com/astral-sh/setup-uv) では一部の条件でキャッシュしてくれないことに気づきました。
幸い元から TypeScript で書かれていたので、すんなり理解することができ、改善する機会をいただくことができました。

- [astral-sh/setup-uv#178](https://github.com/astral-sh/setup-uv/pull/178)

本記事はこれらの経験を踏まえて、 setup 系 GitHub Actions を作る際のキャッシュに関する話をまとめたものです。

## TL;DR

- Setup 系 GitHub Actions はキャッシュ機能が備わっていることが望ましい
  - 雑な 3rd Party Actions ではそもそもキャッシュされていない、または適切な場所にキャッシュされていない場合があるので注意
- キャッシュは **Self-hosted Runner 上で効果抜群**
- キャッシュの保存場所
  - `${{ runner.tool_cache }}` / `$RUNNER_TOOL_CACHE` 以下
  - 階層ルール
    - `<tool-name>/X.Y.Z`
    - `<tool-name>/X.Y.Z/<arch>` (arch は任意)
- 開発するなら **`@actions/tool-cache` を使いましょう**

## Setup 系 GitHub Actions とは

本記事で言及する「Setup 系 GitHub Actions」という言葉についてですが、以下のいずれかを指すことにします。

- [actions org](https://github.com/actions) によって提供されている `setup-*` Actions
  - 任意のバージョンの各ツール (Python, Node.js, その他の CLI ツールなど) を、インストールしてくれる Custom Action のこと
  - 例: actions/setup-node, actions/setup-python
- 上記ツールと似た機能を提供する 3rd Party Actions
  - 例: astral-sh/setup-uv, pollenjp/setup-shfmt

## キャッシュは Self-hosted Runner と相性が良い

よく github.com で利用する Runner として GitHub-hosted runners (`ubuntu-latest` 等) があります。これらは Job を実行する度に新しい VM を割り当てられ、その中で処理を実行します。

一方で、Self-hosted Runner は同一サーバー上で様々な job を実行します。
つまり、最初の Job 実行時にインストールしたツールをキャッシュしておくと、2 度目以降の Job ではキャッシュからツールを利用できるため、無駄な通信を削減し、時間を短縮することができます。

## キャッシュされるケース・されないケース

pollenjp/setup-shfmt の仕様例を見てみましょう。

例えば以下のような workflow (lint.yaml) を考えてみます。

例: pollenjp/install-uv.sh

```yaml
name: Sample Lint
on:
  pull_request:
jobs:
  shellcheck:
    runs-on: ["ubuntu-latest"]
    steps:
      - uses: pollenjp/setup-shellcheck@v1
      - uses: pollenjp/setup-shellcheck@v1
```

これが実行されたときのログは以下です。

[![Image from Gyazo](https://i.gyazo.com/99993dffc33aacc80742b29f7e38095f.png)](https://gyazo.com/99993dffc33aacc80742b29f7e38095f)

同じ `setup-shellcheck` を 2 回実行しているのに、2 回目の実行ではツールのダウンロードが発生していません。

- 1 つ目の setup では最新の shellcheck がダウンロードされ、「適切な場所」に保存・配置されます。
  - > Downloading from https://github.com/koalaman/shellcheck/releases/download/v0.10.0/shellcheck-v0.10.0.linux.x86_64.tar.xz
- 2 つ目の setup では、「適切な場所」に同じバージョンの shellcheck が存在することを検知し、キャッシュを利用するように切り替えます。
  - 後述しますが、この「適切な場所」とは Self-hosted Runner 上では各 Job をまたいで使い回すことのできるパスになっています。

一方、キャッシュが行えていない setup 系 Action はというと、説明するまでも無いかもしれませんが、1 つ目と同様、2 つ目の実行でもツールのダウンロードが発生してしまいます。
尤もキャッシュを意識して書いていない Actions がそんな丁寧にログ出力しているとは限りませんが...

## 具体的なキャッシュの保存場所

先程のセクションでは「適切な場所」という言葉を使いました。

この「適切な場所」は次に示す 2 つの場所をあわせた場所になります。

- キャッシュの保存可能なパス
  - Self-hosted Runner では Job を跨いで利用できる場所
- バージョン付きのツールの階層ルール

### キャッシュの保存可能なパス

キャッシュ用のパスは

`${{ runner.tool_cache }}` または `$RUNNER_TOOL_CACHE`

で提供されており

- GitHub-hosted Runner の場合は `/opt/hostedtoolcache` 配下
- Self-hosted Runner の場合は `<runner-path>/_work/_tool` 配下

のことを指します。 それぞれ [runner context](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#runner-context) や [Default environment variables](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#default-environment-variables) のドキュメントに説明があります。

> The path to the directory containing preinstalled tools for GitHub-hosted runners.

GitHub-hosted Runner は使い回すことがほぼないので `preinstalled tools` という言葉からピンと来ないかもしれませんが、 Self-hosted Runner で言い換えると **「ある Job が終わっても残り続けるディレクトリ」** という事になります。

### バージョン付きのツールの階層ルール

さて、`${{ runner.tool_cache }}`のようにキャッシュ可能なディレクトリが提供されていることはわかりましたが、そのディレクトリの中でツールごとにバラバラなルールで保存されてしまうと統一的な管理が難しくなってしまいます。そこで GitHub Actions はツールごとにバージョンやアーキテクチャに応じた階層ルールをベストプラクティスとして以下のように定めています[^1]。

[^1]: 説明の都合上逆説的な誘導になっていますが、後述する tool-cache がそのようなルールを持っているというだけです。

- `<tool-name>/X.Y.Z`
- `<tool-name>/X.Y.Z/<arch>` (arch は任意)

これを `pollenjp/setup-shellcheck` に当てはめると以下です。

`${{ runner.tool_cache }}/shellcheck/0.10.0/shellcheck`

となり、`shellcheck/0.10.0` というディレクトリルールを守り、一番深い階層に配置された shellcheck が実行ファイルになっています。

## tool-cache を使おう

これらのルールに則った管理は [`@actions/tool-cache`](https://www.npmjs.com/package/@actions/tool-cache) というパッケージが担ってくれます。今までそういうルールがあるかのように話してきましたが、「公式からこういうパッケージが提供されているのだからこれに則ったルールを共通して守りましょう」ということを言いたかっただけです。[^2]

[^2]: 先に tool-cache というツールの使い方を説明するよりもとっつきやすいかと思いこういう説明にしました。

ソースコードは actions/toolkit 配下にサブパッケージとして梱包されています。

<https://github.com/actions/toolkit/tree/main/packages/tool-cache>

特に難解な処理が或るわけではなく、ツールをダウンロードしたり、階層ルールに則ってキャッシュしたり、そこから取り出したりするだけのシンプルなパッケージです。 README やソースコードを見ればどういうメソッドがあるかは大体わかると思います。

実際に正しく利用されいてる 3rd Party Action はたくさんありますので、皆さんの身近なツールで公式の setup 系 Action を提供しているリポジトリがありましたらぜひ中身をみて参考にしてみてください。

※ どうして JavaScript (or TypeScript) で実装されているのかと思われた方がいるかも知れませんが、 GitHub Actions は ubuntu,macos,windows のプラットフォームでの Node.js 環境をサポートしているため制約無く記述することができます。 Shell だとパッケージ配布の時点でどうしても壁がありますからね。

> <https://docs.github.com/en/actions/sharing-automations/creating-actions/about-custom-actions#types-of-actions>
>
> | Type              | Linux | macOS | Windows |
> | ----------------- | ----- | ----- | ------- |
> | Docker container  | o     | x     | x       |
> | JavaScript        | o     | o     | o       |
> | Composite Actions | o     | o     | o       |

もしまだ shell 等でしか workflow を記述したことが無いという方は [actions/typescript-action](https://github.com/actions/typescript-action) のようなテンプレートリポジトリもありますのでぜひご活用ください。

## おまけ: @actions/cache

本記事は [`@actions/tool-cache`](https://www.npmjs.com/package/@actions/tool-cache) について言及しましたがそれとは別に [`@actions/cache`](https://www.npmjs.com/package/@actions/cache) というパッケージもあります。

ちょっと名前が似ていて区別がつきにくいですが、こちらはビルドなどで作成されたファイルを一定期間キャッシュサーバーに保存し、使いたいときに作成済みのものをダウンロードできる優れ者です。

GitHub-hosted Runner だとこちらのほうが良くお世話になるかもしれません。

また、`@actions/cache` の機能を 1 つの custom action として利用可能な [actions/cache](https://github.com/actions/cache) というものもありますので上手く使い分けられるようになると便利です。
