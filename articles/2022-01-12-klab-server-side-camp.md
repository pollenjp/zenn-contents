---
title: "KLab の Server Side Camp に参加してきました"
emoji: "🖥️"
type: "tech"
topics: ["ポエム"]
published: false
---

## はじめに

先日, 株式会社KLab(クラブ) さんの KLab Server Side Camp 第1回 (5日間) に参加してきました.
私は個人の遊び程度でしかサーバサイドを触ったことが無かったため, 実務をなさっている方のそばで学び質問できる環境はとてもありがたかったです.
というわけで本稿ではその感想でも書いていこうと思います.
軽い振り返りなので技術的な内容は少なめです^^

[KLab Server Side Camp 第2回](https://klab-hr.snar.jp/jobboard/detail.aspx?id=KAmfKyX-LPE) の募集が 1/24(月) 23:59 まで行われているので興味を持った方はぜひ. (日程：3/10(木)～16(水) ※平日5日間)

## KLab Server Side Camp とは

[第1回応募ページ (受付終了)](https://klab-hr.snar.jp/jobboard/detail.aspx?id=dOPYn2c4CEE)

> サーバサイド特化型インターン「KLab Server Side Camp」 (第1回)
> KLab Server Side Camp (クラブサーバサイドキャンプ) は,サーバサイド特化型の技術系インターンです.
> 本イベントの為にオリジナルで自社開発したスマホゲーム (音ゲー) を題材に,ゲームアプリの中でサーバサイドの技術がどのように使われているかを,講義形式で進めつつ実際に課題にも取り組み手を動かしてもらいながら経験を積むことができるサーバサイド特化型のインターンです.
> Python開発経験やGithub使用経験等は応募時に必要なスキルとなりますが,ゲーム開発経験の有無は問いません.
> むしろ,今まで研究活動や趣味でPythonを使ってきたけれど,それが企業でどのように活かせるのかまだ想像しきれない学部生や院生の皆さん等にぜひおすすめしたいインターンです.
> ※尚,今回の為にオリジナルで開発したスマホゲーム (音ゲー) は,社内のエンジニアさんはもちろんクリエイティブ職の皆さんに,イラスト,UI,アートディレクション等をお願いして制作したものです. 参加される方にはぜひゲームそのものも楽しんでもらえればと思います!

一言でいえば, サーバサイドに興味はあるけど経験のないそこそこPython経験者が無料でスマホゲーのサーバサイド講義を受けられるキャンプになります (オンライン).

自分は以下の `@methane` さんのツイートを見て第1回に応募しました.

@[tweet](https://twitter.com/methane/status/1438835441868828673?s=20)

## 1-2 日目 (2021/12/27 - 2021/12/28)

各自参加者の自己紹介から始まり, その後のお昼休みで早速用意していただいた音ゲーをプレイしました.
自分はプロセカがリリースされた時から音ゲーを始めた初心者です.
普段は4本の指しか使わないので今回の6本指の音ゲーに結構苦戦しました^^;

1日目の午後は本キャンプで主に使用する GitHub Codespaces, MySQL, SQLAlchemy, FastAPI, pydantic の最低限の説明を受けました.
以下の雛形をベースに簡易なユーザ登録・更新処理を実装しました.

<https://github.com/KLabServerCamp/gameserver>

2日目にはある程度の知識がつき全体像も見えてきたため, マルチプレイ用のルームAPIの実装を行うという課題が与えられました.
Request, Response の仕様は既に決まっていますが, 「どのようなデータベースのテーブルにし」「どのような処理で実現するか」にはある程度の自由度があり, ちょうどよい難易度の課題でした.
この Room API の完成が本キャンプでの目標になります.

## 冬休み・お正月 (2021/12/29-2022/01/04)

最終目標がある程度見えてきたところで次はお正月明けになります.

自分はこの期間を使って普段Python開発に用いる開発環境をいくつか追加しました.
例えば poetry, black, flake8, autoflake8, isort, mypy, pytest, nox 等です (既に導入済みのものを含む).

休み中は年末年始であることもあり, 念のため GitHub Codespaces を停止することになりました.
そのため個人での Codespaces の利用 or ローカルでの開発のみが可能でした.
Ubuntu/Linuxを普段から使っていることもありローカルでの開発環境はすぐに整いました.
一方で GitHub Codespaces は事前申請が必要そうなのでそれを待たねばならず個人での利用はまだ試せていません.

@[tweet](https://twitter.com/polleninjp/status/1476605747375251459?s=20)

2022/01/12現在
[![Image from Gyazo](https://i.gyazo.com/c16ab27633900b84c970c186f5fbddf5.png)](https://gyazo.com/c16ab27633900b84c970c186f5fbddf5)

> Sign up for the Codespaces beta
> Join the Codespaces beta waitlist to get a cloud development environment you can access from anywhere. We’d love to hear your feedback while you’re trying this new feature.
> You’re already on the waitlist for Codespaces! We’ll notify you when we’ve enabled it on your account. Make sure your primary email address is up-to-date so we can get a hold of you.

まぁローカル開発環境は整ったので8割りほど実装を終えて新年を迎えることができました.

## 3-4 日目 (2022/01/05-2022/01/06)

Room API 実装に本格的に取り組む予定となっていたこの日に自分は雑なミスでデバッグに半日ほど取られていました...
メンターさんのお時間も頂戴してしまい反省しました.
やはりちゃんとテスト書いてから駆動していけばよかったなと.

残った時間は SQLAlchemy や pydantic のドキュメントを呼んだりしていました.
特に pydantic は dataclasses.dataclass の上位互換な匂いがするのでもう少し調べて積極的に使っていきたいと感じました.

終盤は参加者同士で部屋を立て合ってデバッグしてました.

## 5 日目 (2022/01/07)

さて, 最終日の5日目ですが.........

寝坊しました (10:30集合のところ11:18起床)

![Image from Gyazo](https://i.gyazo.com/87c24f79a1173c8a9829c23117a9d084.png)

実はその日の懇親会の配達物が届いたため起きることができました. 寝坊防止策として優秀かもしれない...

さて, 最終日は最後の調整と成果報告をまとめたスライドを作り最後に互いに発表し合いました.
