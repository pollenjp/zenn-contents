---
title: "GitHub Pages (Jekyll) のレンダリング結果をローカルで確認する"
# emoji: "😸"
type: "tech"
topics: ["githubpages", "docker", "jekyll"]
published: false
---

本記事は [UT-virtual Advent Calendar 2021](https://qiita.com/advent-calendar/2021/ut-virtual) の22日目の投稿です.

## 前置き (ほぼ身内向け)

※ 技術的内容は次のセクションからですのでどうぞ読み飛ばしてください.

(2021/12/21 23:30:00)
すっかすかの UT-virtual Advent Calendar 2021 を見ながらとりあえず埋めるかーというテンションで埋めたので何を書こうと頭を悩ます
VRサークルではあるものの私個人は直近2年ほどVR開発してない... ＿(:3｣∠)＿

今年のサークル内では

- 2021年序盤ではいつも通り Unreal Engine に関する雑な部内LT会したり (UELTと勝手に呼称)
- 去年に引き続き GitHub Organization の管理したり (まだたいしたことやれてない)
- DirectXの技術書(『DirectX 12の魔導書』)を輪読したり (継続中かつ現在参加者私含めて二人...ﾋﾟｴﾝ)

してたけど, 日々の忙しさを理由にどれも記事にできるほどのキリ良さがない...(´◉ω◉` )

アドカレの登録をしたときは部の GitHub Organization をどう管理しているかを書こうとか考えていたけど, 管理サーバ等を独自に作っていたら完成する前に今日 (投稿日) が来てしまった...
※これに関してはいずれ記事にするかも

(2021/12/21 23:56:00)
あー何書こうかなぁ

数ヶ月前に GitHub Pages をローカルでレンダリングする方法調べたことがあったからそれにしよう(唐突).

締切駆動開発において締切を破るプロフェッショナルである私は, 締切までに管理サーバが完成しないことを見越してアドカレの予定欄にちゃんと "GitHub Organization 運用の話 or GitHub Pagesでできること" って書いていた...ｴﾗｲ...ｴﾗｲ

ってなわけで GitHub Pages の関連話をします.

以下から本題です.

## GitHub Pages

<https://pages.github.com/>

面倒なので全く知らない人向けの説明は省きますが, 簡単に言えばGitHubのレポジトリに対応して静的なWebページをホスティングしてくれるサービスです.
自身のGitHubのユーザ名に対して `ユーザ名.github.io` のドメインでアクセスできます.
例えば, 直接HTML, CSS, JS等をレポジトリに配置することで静的なWebページを作ることができます.

## Jekyll

- [Jekyll • シンプルで、ブログのような、静的サイト | プレーンテキストを静的サイトやブログに変えましょう](http://jekyllrb-ja.github.io/)
- [Jekyll を使用して GitHub Pages サイトにコンテンツを追加する - GitHub Docs](https://docs.github.com/ja/pages/setting-up-a-github-pages-site-with-jekyll/adding-content-to-your-github-pages-site-using-jekyll)

一方, GitHub Pages は Jekyll で動いているためそれを利用することで様々な恩恵を得ることができます.
例えば,

- テンプレートを用いたデザイン性の統一
- 変数による GitHub Pages 上の情報の埋め込み
- マークダウンによる記述

等が上げられます.

サンプルとして以下のプロジェクトを用意しています. (レポジトリの内容が変わらないようにコミットIDで指定しています.)

<https://github.com/pollenjp/sandbox-github-pages/tree/5994da3b4e687a399c8954140ed49a68c512425f>

このプロジェクトでは `docs/` 以下をレンダリングするように指定しています.

例えば, `index.md` は トップページである `index.html` にレンダリングされ, 以下のように表示されます.

`index.md` : <https://github.com/pollenjp/sandbox-github-pages/blob/5994da3b4e687a399c8954140ed49a68c512425f/docs/index.md>

[![Image from Gyazo](https://i.gyazo.com/3e3396a5475185dfa83db16d1622e1f9.png)](https://gyazo.com/3e3396a5475185dfa83db16d1622e1f9)

## ローカルでレンダリング結果を確認したい

先の結果は GitHub 上で実際にレンダリングされたものの画像を添付していますが, 確認する度に GitHub にコミットし `github.io` のURLにアクセスすることでレンダリング結果を確認するのはとても億劫ですし, そもそも確認もせずに本番の環境に反映させるのは言語道断です.

※ ちなみに私は最初これやってて, sandbox用のプロジェクトではありますが微調整する度にコミットして GitHub Pages のレンダリング・反映を待つ必要がありました.

そこで, GitHub Pages で使われているレンダリングサーバと同じサーバをローカルで起動してそれでレンダリングすればよいと考えいろいろ調べてオレオレな起動コマンド (?) を確立しました.

## 先に結論

以下の `Makefile` を記述し `make run` で実行しています.

<https://github.com/pollenjp/sandbox-github-pages/blob/5994da3b4e687a399c8954140ed49a68c512425f/Makefile>

```Makefile
ROOT := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))

JEKYLL_DOCKER_IMAGE := "jekyll/jekyll"

CONTAINER_REPO_DIRPATH := /srv/jekyll/repo
PAGES_PATH := docs

HOST_PORT := 4000
CONTAINER_PORT := 4000

.PHONY: run
run:
	docker run \
		--rm \
		--volume "${ROOT}:${CONTAINER_REPO_DIRPATH}" \
		--interactive \
		--tty \
		--publish ${HOST_PORT}:${CONTAINER_PORT} \
		${JEKYLL_DOCKER_IMAGE} \
		bash -euxc '\
			echo "Generating config for local run" && \
			local_config_path=/local_run_config.yml && \
			echo "github:"                                             > $${local_config_path} && \
			echo "  url: \"http://$$(hostname -i):${CONTAINER_PORT}\"" >> $${local_config_path} && \
			echo "bundle install and exec" && \
			cd ${CONTAINER_REPO_DIRPATH} && \
			bundle install && \
			bundle exec jekyll serve \
				--force_polling \
				--host $$(hostname -i) \
				--trace \
				--config ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH}/_config.yml,$${local_config_path} \
				--source ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH} \
				--layouts ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH}/_layouts \
				--destination ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH}/_site \
		'

.PHONY: help
help:
	docker run --rm -it ${JEKYLL_DOCKER_IMAGE} jekyll build --help
```

以降で要点ごとに解説していきます.

## Jekyll Docker Image

- <https://hub.docker.com/r/jekyll/jekyll/>
- <https://github.com/envygeeks/jekyll-docker/blob/b8d394ad2a6b13fbbce23b6eea94cd9902f492e8/README.md>

Jekyll は Docker Image を提供しているためこれをそのまま使わせてもらいます.

## 起動準備 (bundle install)

`docker run` でこの Image からコンテナを起動するのですが, jekyll 実行する前に今回は [minimal](https://github.com/pages-themes/minimal) テンプレートを使用するため README.md 記載されている通り, `Gemfile` の作成・ `bundle install` の実行をします.

<https://github.com/pollenjp/sandbox-github-pages/blob/5994da3b4e687a399c8954140ed49a68c512425f/Gemfile>

```Gemfile
source 'https://rubygems.org'

gem 'github-pages', group: :jekyll_plugins
```

## サーバ起動 (bundle exec jekyll serve)

`Makefile` 一部抜粋.

```Makefile
bundle exec jekyll serve \
    --force_polling \
    --host $$(hostname -i) \
    --trace \
    --config ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH}/_config.yml,$${local_config_path} \
    --source ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH} \
    --layouts ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH}/_layouts \
    --destination ${CONTAINER_REPO_DIRPATH}/${PAGES_PATH}/_site \
```

- `--force_polling` : ファイルの内容が変更されたときそれを検出し自動で再レンダリングする.
- `--host` : ホストを指定していますが, hostname コマンドで Docker Container 起動時のホストを取得しています. ( `Makefile` 内でシェル実行時に変数展開等をする場合は `$$` とする.)
- `--trace` : エラー発生時にトレースを出力する.
- `--config` : 設定を指定する際にファイルパスを渡す. 複数ある場合は `,` 区切りで渡す. (後述)
- `--source` : レンダリング対象のディレクトリを指定.
- `--layouts` : `_layouts` ディレクトリを用意している場合はこれで渡す.
- `--destination` : レンダリング結果を出力ディレクトリを指定.

## 問題点

template html 内で使われている `site.github.url` の変数がデフォルトの値になってしまいます.

例えば, 以下の例 (`{{ site.github.url }}{{ a_page.url }}`) では posts (jekyllにおける記事群のようなもの) から各記事オブジェクトを取得しそのページへのリンクを生成しています.
その際に `{{ site.github.url }}` を使用して root までの URL を取得し, 相対パス (`{{ a_page.url }}`) とつなげています.

※ 実際には `{{ site.github.url }}` の部分は必要なく相対パスだけで十分です. むしろそっちの方が良いですがあくまで説明のため無駄に書いています.

<https://github.com/pollenjp/sandbox-github-pages/blob/5994da3b4e687a399c8954140ed49a68c512425f/docs/index.md>

```index.md
## Welcome to GitHub Pages

Hoge Hoge

## Pages

<ul>
  {% for a_page in site.html_pages %}
    {% if a_page.title != page.title %}
      <li>
        <a href="{{ site.github.url }}{{ a_page.url }}">{{ a_page.title }}</a>
      </li>
    {% endif %}
  {% endfor %}
</ul>

## Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ site.github.url }}{{ post.url }}">
        {{ post.date | date: "%Y-%m-%d" }} {{ post.title }}
      </a>
    </li>
  {% endfor %}
</ul>
```

`site.github.url` のデフォルト値はgitの追跡情報から取得したいリモートレポジトリURLであるため, ローカルでレンダリングしたときに上記のリンクが異なるページをさしてしまいます.
※ リモートレポジトリURL を取得したい際は `site.github.repository_url` を使います.

そのため, この値をローカルで起動するサーバのURLに変更する必要があります.

まず, 設定を渡すために設定ファイルを作成します.

```Makefile
			local_config_path=/local_run_config.yml && \
			echo "github:"                                             > $${local_config_path} && \
			echo "  url: \"http://$$(hostname -i):${CONTAINER_PORT}\"" >> $${local_config_path} && \
```

この部分によってコンテナ内のルートディレクトリに `local_config_path.yml` というファイルが作成されます.

```local_config_path.yml
github:
  "http://containerhost:port"
```

※ containerhost は実行時のコンテナのホスト, port は指定したポートに読み替えてください.

これを, `bundle exec jekyll serve` 時の `--config` オプションで渡すことで設定を上書きすることができます.

## レンダリング結果

以上を元にローカルで `make run` すると, `bundle install` 等の処理が走り最終的には以下のようなログが見えます.

```sh:log
+ bundle exec jekyll serve --force_polling --host 172.17.0.3 --trace --config /srv/jekyll/repo/docs/_config.yml,/local_run_config.yml --source /srv/jekyll/repo/docs --layouts /srv/jekyll/repo/docs/_layouts --destination /srv/jekyll/repo/docs/_site
Configuration file: /srv/jekyll/repo/docs/_config.yml
Configuration file: /local_run_config.yml
   GitHub Metadata: No GitHub API authentication could be found. Some fields may be missing or have incorrect data.
            Source: /srv/jekyll/repo/docs
       Destination: /srv/jekyll/repo/docs/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 0.132 seconds.
 Auto-regeneration: enabled for '/srv/jekyll/repo/docs'
    Server address: http://172.17.0.3:4000
  Server running... press ctrl-c to stop.
```

ここでは

```txt
    Server address: http://172.17.0.3:4000
```

とあるように <http://172.17.0.3:4000> にアクセスすればページが表示されます.

[![Image from Gyazo](https://i.gyazo.com/4890e8e635faba2ddf60cbcc4b3684bd.png)](https://gyazo.com/4890e8e635faba2ddf60cbcc4b3684bd)

## 最後に

私は Jekyll も Ruby もほとんど触ったことない素人ですので, ここに書いてあることを鵜呑みにせず"最新の"公式ページ等で確認すると良いでしょう.

また, 出過ぎたことは書かないように気を付けたつもりではありますが, もし誤り等がありましたらコメントや記事のプルリク等宜しくお願いいたします.

## 参考

- GitHub Pages
  - <https://pages.github.com/>
- jekyll
  - [Jekyll • シンプルで、ブログのような、静的サイト | プレーンテキストを静的サイトやブログに変えましょう](http://jekyllrb-ja.github.io/)
  - [Jekyll を使用して GitHub Pages サイトにコンテンツを追加する - GitHub Docs](https://docs.github.com/ja/pages/setting-up-a-github-pages-site-with-jekyll/adding-content-to-your-github-pages-site-using-jekyll)
  - <https://hub.docker.com/r/jekyll/jekyll/>
  - <https://github.com/envygeeks/jekyll-docker/blob/b8d394ad2a6b13fbbce23b6eea94cd9902f492e8/README.md>
  - [minimal](https://github.com/pages-themes/minimal)
- ブログ
  - <https://alpha3166.github.io/blog/20190413.html>
  - <https://qiita.com/shifumin/items/8d5d26dfa18d4b62d873>
