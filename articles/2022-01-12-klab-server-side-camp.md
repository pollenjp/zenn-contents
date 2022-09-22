---
title: "KLab の Server Side Camp に参加してきました"
emoji: "🖥️"
type: "idea"
topics: ["ポエム"]
published: true
---

## はじめに

先日, 株式会社KLab(クラブ)さん (以降敬称略) の KLab Server Side Camp 第1回 (5日間) に参加してきました.
私は個人の遊び程度でしかサーバサイドを触ったことが無かったため, 実務をなさっている方のそばで学び質問できる環境はとてもありがたかったです.
というわけで本稿ではその感想でも書いていこうと思います.
軽い振り返りなので技術的な内容は少なめです^^
また, 久々に記事書くので文章がおかしくなっているかも...

追記 (2021/01/19)
メンターの methane さんによるKLab技術ブログ (「DSAS開発者の部屋」) は以下のツイートからどうぞ.

@[tweet](https://twitter.com/methane/status/1481929104614379523?s=20)

## 告知

[KLab Server Side Camp 第2回](https://klab-hr.snar.jp/jobboard/detail.aspx?id=KAmfKyX-LPE) の募集が 1/24(月) 23:59 まで行われているので興味を持った方はぜひ!!

@[tweet](https://twitter.com/oktillion27/status/1469249287863545856?s=20)

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

## スケジュール

### 1-2 日目 (2021/12/27 - 2021/12/28)

各自参加者の自己紹介から始まり, その後のお昼休みで早速用意していただいた音ゲーをプレイしました.
自分はプロセカがリリースされた時から音ゲーを始めた初心者です.
普段は4本の指しか使わないので今回の6本指の音ゲーに結構苦戦しました^^;

1日目の午後は本キャンプで主に使用する GitHub Codespaces, MySQL, SQLAlchemy, FastAPI, pydantic の最低限の説明を受けました.
以下の雛形をベースに簡易なユーザ登録・更新処理を実装しました.

<https://github.com/KLabServerCamp/gameserver>

2日目にはある程度の知識がつき全体像も見えてきたため, マルチプレイ用のルームAPIの実装を行うという課題が与えられました.
Request, Response の仕様は既に決まっていますが, 「どのようなデータベースのテーブルにし」「どのような処理で実現するか」にはある程度の自由度があり, ちょうどよい難易度の課題でした.
この Room API の完成が本キャンプでの目標になります.

### 冬休み・お正月 (2021/12/29-2022/01/04)

最終目標がある程度見えてきたところで次はお正月明けになります.

自分はこの期間を使って普段Python開発に用いる開発環境をいくつか追加しました.
例えば poetry, black, flake8, autoflake8, isort, mypy, pytest, nox 等です (既に導入済みのものを含む).

休み中は年末年始であることもあり, 念のため GitHub Codespaces サーバを停止することになりました.
そのため個人での Codespaces の利用 or ローカルでの開発のみが可能でした.
Ubuntu/Linux機を持っていることもありローカルでの開発環境はすぐに整いました (poetry + MySQL(Docker) ).
一方で GitHub Codespaces は事前申請が必要そうなのでそれを待たねばならず個人での利用はまだ試せていません.
とはいうものの裏では devcontainer が動いているのでローカルで Docker を使えばすぐに同じ環境が構築できると思います.

@[tweet](https://twitter.com/polleninjp/status/1476605747375251459?s=20)

2022/01/12現在
![Image from Gyazo](https://i.gyazo.com/c16ab27633900b84c970c186f5fbddf5.png)

> Sign up for the Codespaces beta
> Join the Codespaces beta waitlist to get a cloud development environment you can access from anywhere. We’d love to hear your feedback while you’re trying this new feature.
> You’re already on the waitlist for Codespaces! We’ll notify you when we’ve enabled it on your account. Make sure your primary email address is up-to-date so we can get a hold of you.

まぁローカル開発環境は整ったので8割りほど実装を終えて新年を迎えることができました.

### 3-4 日目 (2022/01/05-2022/01/06)

Room API 実装に本格的に取り組む予定となっていたこの日に自分は雑なミスでデバッグに半日ほど取られていました...
メンターさんのお時間も頂戴してしまい反省しました.
やはりちゃんとテスト書いてから駆動していけばよかったなと.

残った時間は SQLAlchemy や pydantic のドキュメントを呼んだりしていました.
特に pydantic は dataclasses.dataclass の上位互換な匂いがするのでもう少し調べて積極的に使っていきたいと感じてます.

終盤は参加者同士で部屋を立て合ってデバッグしてました.

4日目の途中でKLabのエンジニアさん座談会 (トークセッション？) の時間を設けていただき, KLabの雰囲気を感じることができました.

### 5 日目 (2022/01/07)

さて, 最終日の5日目ですが.........

寝坊しました (10:30集合のところ11:18起床)

![Image from Gyazo](https://i.gyazo.com/87c24f79a1173c8a9829c23117a9d084.png)

実はその日の午前に懇親会の配達物が届くとのことだったのでそのチャイムのおかげで目を覚ましました (O_O)

さて, 最終日は成果物の調整, 成果報告スライド作り, 成果発表, 懇親会 といった流れになります.
成果発表では講師の方からテーブル設計などのアドバイスコメントをもらいつつ, また周りの参加者たちがどんな目線で参加していたのかという点を知ることができ新鮮でした.

懇親会ではKLabエンジニアさん達から「エンジニアオタク集団臭」が漂っていてとてもよい機会になりました.

## 完成物 (Room API)

本キャンプのメインになる Room API 部分を少しご紹介します.
コードの解説をすると長くなりそうなのでどんなことができるようになったのかを画像で説明します.
雑にスクロールして雰囲気を感じてください^^

これ以外にも細かい機能がありますが, 気になる方はソースを見てみてください.
TableName 用のクラスを作成する等好き勝手なコードの書き方をしていますがご了承ください.
(MySQLのクエリ文を直接実行するためあえて sqlalchemy の `connection.execute` のみを利用しています.)

- <https://github.com/pollenjp/klab-gameserver-forked>
- <https://github.com/pollenjp/klab-gameserver-forked/blob/b55230e95a2f9e5daea8e4ada2b908ed0297d2d2/app/room_model.py>

### 環境

今回の音ゲーのクライアントアプリは同一PCで複数起動しても同時に扱えるユーザは一人だけだったため複数のPC + Windows Sandbox を用意して起動してみました.

| ユーザ名 | 物理PC | OS |
|:--|:--|:--|
| pollenjp1 | PC1 | Windows10 Home |
| pollenjp2 | PC2 | Windows10 Pro |
| pollenjp3 | PC2 | Windows Sandbox |

### Create Users

- pollenjp1-3 のユーザを作成

![Image from Gyazo](https://i.gyazo.com/61f05a86cb55a3c627496313087d501b.png)
![Image from Gyazo](https://i.gyazo.com/6172b770b4d488029296868e6f88e521.png)

### Create Room

- マルチプレイ用Room作成
- pollenjp1 がホスト

![Image from Gyazo](https://i.gyazo.com/0ae964471cd90df1c1620396d8544385.png)

### Join Room

- pollenjp2 が先ほど作成した部屋に入ります.

![Image from Gyazo](https://i.gyazo.com/fcc441e4296ec8ff07e4dce8bc47abb1.png)

### Join Room (RoomFull)

- 今回はテストのため同時接続数を2に制限しているため3人目の入室を拒んでいます.
- pollenjp3 が既に満室のルームへ参加しようとして拒まれています.

![Image from Gyazo](https://i.gyazo.com/c66f1d280ece2a92dc68df95adb45d66.png)

### Start

- Host 側にのみ「ライブ開始」ボタンが表示されているのでホストがボタンを押しタイミングでお互いのゲームが開始します.

![Image from Gyazo](https://i.gyazo.com/57511744d385981e4bc350dc329212a0.png)

### Result

- 同じルームにいる人のスコア等を表示します.

![Image from Gyazo](https://i.gyazo.com/2c1e96c1378700284d5f338b85fe058e.png)

以上

## 全体を通しての感想

### 音ゲー

プロセカ以外の音ゲーに初めて取り組み, キーボードタイプに慣れるのが難しかったですが, だいぶ慣れてくると楽しくプレイすることができました.
自分のボカロ好きかつ音ゲースキルレベルがマッチしたという意味で『コタエサガシコード』を何度もプレイしていました. いい曲ですね^^

@[tweet](https://twitter.com/magatsu_klab/status/1188410803268907008?s=20)

また, 参加者に人間を辞めた音ゲーマーがいてビビりました.
その場で渡された音ゲーにいち早く慣れ, 最高評価 (?) をたたき出していました.
恐るべし...

### 技術面

課題の難易度が個人的にはちょうどよくメインの処理を記述しつつも余裕を持ってドキュメント等を漁ることができました.

また, 個人的に一番良かった点は **実務に対する知見を持つ講師・メンターの方にいろんな質問ができる点** です.

個人での開発は書籍やブログ記事等で学ぶことができますが「実務ではどのようにして対処するのか」という視点の回答を得るのは結構難しいものがあります.
(オンラインで活動しているコミュニティ等で質問することが可能な場所もあるのですが結構気を使いがち)
そんな中で今回はバンバン質問することが許された環境をいただいたため普段自分が気になっていたことを質問する機会になりました.

また, 仕様などで理解が曖昧な箇所を確認の意味をこめてつぶやくことで理解が深まる他に派生知識を教えていただける場面もありました.

### 会社を知る

実のところ本キャンプを参加する前までは KLab のことをあまり認知しておらず,Wikipediaで社名の呼び方を調べる等からスタートしたくらいです.

自分の過去のドメインからゲーム会社は縁が遠そうと勝手な印象を持っていたのですが,裏で使われている技術一つ一つに目を向けるとそれが勘違いであることに気付きました.

また, [KLablog](https://www.klab.com/jp/blog/) や [klab_teck YouTube Channel](https://www.youtube.com/channel/UCBMRcOQHH_kJ5o0SP6AfwHg) 等を漁ってみると見つかるエンジニアオタク溢れる記事が結構面白かったです.

本キャンプを通じて KLab という会社を知り社内の良い雰囲気をつかむことができとても充実した5日間だったと思います.

## 最後に

とまぁつらつら書いてきたわけですが自分が記事を書く力量がないことを改めて実感しましたね...
Private な Scrapbox に雑にメモしまくるようになってから Public にする文章を書かなくなり (ry (言い訳)

それはさておき KLab Server Side Camp ははじめに述べた通り Pythonある程度書いたことあるけど実務に踏み出したことが無い人が気軽に挑戦できる内容だったので興味がある人は第2回 or それ以降の募集状況を追ってみると良いかもしれません.

他にも KLab Expert Camp というものも何度か開催されているためこちらもまとめてチェックしてみるのをお勧めします.

ではまた (@'ω'@)ノシ

[#KLabExpertCamp OR #KLabServerSideCamp (Twitter Search)](https://twitter.com/search?q=%23KLabExpertCamp%20OR%20%23KLabServerSideCamp&src=typed_query)

@[tweet](https://twitter.com/oktillion27/status/1480873517835255809?s=20)
