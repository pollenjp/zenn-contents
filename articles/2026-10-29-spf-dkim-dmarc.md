---
title: "SPF・DKIM・DMARC がメールのどこを見ているのか、封筒と便箋から整理してみた"
emoji: "📮"
type: "tech"
topics: ["dns", "spf", "dkim", "dmarc", "googleworkspace"]
published: true
---

## はじめに

「SPF を設定してください」「DMARC のレコードを足してください」。Google Workspace のヘルプによく出てくる指示だ。言われたとおり DNS に文字列を貼れば設定は終わるが、3 つがそれぞれメールのどこを見て、何を確かめているのかは、ふわっとしたままになりがち。それだと、メールが届かなくなったときに手が出ない。

これを機会に整理しようと思い、自分用のメモも兼ねてざっくりまとめる。同じ話を、約 13 分の解説動画にもした。

https://www.youtube.com/watch?v=07GTwfhbKsQ

古いメールサーバーから 1 人ずつ移すあいだ (分割配信) にハマる所は、[続きの記事](https://zenn.dev/pollenjp/articles/2026-10-30-split-delivery) に分けた。

:::message
2026-10-29 時点の情報。画面の名前や手順は変わることがあるので、設定するときは Google のヘルプも見てほしい。ドメインと IP アドレスは説明用。
:::

## TL;DR

- メールの差出人は 2 か所にある。サーバー同士が使う**封筒 (MAIL FROM)** と、人が見る**便箋 (From:)**。どちらも送り手が自由に書ける
- SPF は封筒の差出人のドメインの名簿で接続元を、DKIM は中身の署名を確かめる。どちらも From: は見ていない
- **DMARC が、SPF か DKIM で確かめたドメインと From: が揃っているかを見る**。`p=none` で送り手を洗い出してから強める
- Google Workspace では SPF と DKIM (2048 ビット) を整え、48 時間以上空けて DMARC を `p=none` で入れる

## 差出人は 2 か所ある

サーバー同士のやり取り (SMTP) をのぞくと、差出人が 2 回出てくる。

```text
C: MAIL FROM:<bounce@example.co.jp>      ← 封筒の差出人
C: RCPT TO:<c@example.com>
C: DATA
C: From: お知らせ <info@example.co.jp>   ← 便箋の差出人
```

`MAIL FROM` は配達のための封筒で、届かなかったときの返り先になる。`From:` は便箋の頭で、メールソフトが差出人として見せるのはこっち。Bcc や外部の配信サービスのように、封筒と便箋が食い違うのはふつうのことで、おかしな話ではない。

厄介なのは、1982 年生まれの SMTP には、どちらの差出人も確かめる仕組みが無いところ。受け手に確かに分かるのは、接続してきた IP アドレスだけ。そこで後から、ドメインの持ち主が DNS に「証拠」を置き、受け手がメールのどこかと照らす仕組みが 3 つ足された。

![1 通のメールの 3 か所 (封筒・便箋の頭・署名) と、SPF・DKIM・DMARC がそれぞれ見る場所](/images/2026-10-29-spf-dkim-dmarc/00-three-checks.png)

## SPF は封筒の差出人の名簿

SPF は、「このドメインを封筒の差出人にして送ってよいサーバーは、これだけ」という名簿を DNS に置く仕組み。受信サーバーは MAIL FROM のドメインの TXT レコードを引き、接続元の IP アドレスが載っているかを見る。

```text
v=spf1 ip4:203.0.113.25 include:_spf.google.com ~all
```

左から順に照らし、最初に当たったところで決まる。接続元が 203.0.113.25 か Google のサーバーなら pass、どちらでもなければ最後の `~all` に当たって softfail。

壊しやすい決まりも 2 つある。レコードは 1 つのドメインに 1 つだけ (2 つあると permerror)。`include`・`a`・`mx` のような DNS を引く項目は、入れ子も含めて 10 回まで。

最後は Google のヘルプが勧める `~all` にして、最終的な扱いは DMARC に任せるのがよさそう (`-all` だと、DMARC で判断する前に断る受け手がいる)。

弱点は 2 つ。転送されると接続元が転送のサーバーに変わって通らなくなることと、人が見る From: を見ていないこと。

## DKIM は中身に付ける署名

DKIM は、送信サーバーが秘密鍵でメールに電子署名を付け、受け手が DNS の公開鍵で確かめる仕組み。中身を書き換えると署名が合わなくなる。

公開鍵は `<セレクタ>._domainkey.<ドメイン>` に置く。Google Workspace の既定のセレクタは `google` なので、`google._domainkey.example.co.jp`。

中身を書き換えずに転送すれば署名は残るので、転送に弱い SPF を補える。逆に、件名に `[会員]` を足すメーリングリストのように、途中で中身を書き換えると壊れる。

ただ、DKIM の pass で分かるのは「d= のドメインが署名し、それから中身が変わっていない」ことだけ。d= は From: と同じでなくてよく、配信サービスが自分のドメイン (d=example.net) で署名しても pass する。

## DMARC が From: に結びつける

DMARC の合格は、次のどちらかを満たすこと。

- SPF が pass し、MAIL FROM のドメインが From: と揃っている
- DKIM が pass し、d= のドメインが From: と揃っている

揃い方は既定でゆるく (relaxed)、mail.example.co.jp と example.co.jp のように同じ組織のドメインなら揃っているとみなす。

![DMARC は、合格した SPF か DKIM のドメインが From: と揃っているかを見る](/images/2026-10-29-spf-dkim-dmarc/06-dmarc-decision.png)

設定は `_dmarc.example.co.jp` に置く TXT レコード。

```text
v=DMARC1; p=none; rua=mailto:dmarc@example.co.jp
```

`p=` は不合格のメールをどう扱ってほしいかで、none (見るだけ)・quarantine (迷惑メールへ)・reject (断る) の 3 つ。`rua=` に集計レポートが届くので、問い合わせのフォームや複合機のような、忘れていた送り手が見えてくる。

いきなり reject にすると、そうした送り手のメールがある日から届かなくなる。`p=none` でレポートを見ながら送り手を整え、quarantine、必要なら reject と強めるのが定石。reject は全員のゴールではなく、2026 年 5 月の新しい規格 RFC 9989 は、メーリングリストに投稿するかもしれない人がいるドメインは reject を出すべきでない、としている。

※ RFC 9989 で `pct=` (方針を当てる割合) は無くなったが、Google のヘルプはまだ `pct=` で少しずつ広げる手順を案内している。

## Google Workspace でやること

![Google Workspace での設定の順番。SPF と DKIM を整え、48 時間以上たってから DMARC を p=none で始める](/images/2026-10-29-spf-dkim-dmarc/08-gws-steps.png)

1. **MX**: `smtp.google.com` (優先度 1) の 1 本にする
2. **SPF**: `v=spf1 include:_spf.google.com ~all`。ほかの送り手は、同じ 1 つのレコードに書き足す
3. **DKIM**: 管理コンソールの [アプリ] → [Google Workspace] → [Gmail] → [メールの認証] で 2048 ビットの鍵を作り、`google._domainkey` の TXT レコードを DNS に足してから [認証を開始] を押す
4. **DMARC**: SPF と DKIM で認証されるようになってから 48 時間以上空けて、`p=none` で入れる

SPF と DKIM はどちらが先でもいい。管理コンソールで触るのは DKIM だけで、ほかは DNS の仕事なので、DNS を誰が編集できるかを最初に確かめておくと楽。

Gmail も 2024 年 2 月から、個人の Gmail 宛てに送る人に SPF か DKIM を求めていて、2025 年 11 月からは守らないメールへの措置を強めているとのこと。

## おわりに

3 つの仕組みは、どれも DNS に置いた証拠を受け手が照らす、同じ形をしている。SPF は封筒の差出人と接続元、DKIM は署名と中身、DMARC は人が見る From:。どこに何の証拠を置き、メールのどこと照らすのかが分かると、ヘルプの手順の 1 行 1 行が何のためにあるのかが見えてくる。

自分のドメインの SPF が 2 つになっていないか、`mx` に頼っていないか、一度 `dig` で見ておきましょうね。

参考になれば幸い。誤りがあればコメントで教えてほしい。

## 次の記事

古いメールサーバーから Google Workspace へ 1 人ずつ移すあいだ (分割配信) に、SPF・DKIM・DMARC がどうこじれるかを、架空の会社の例で追った。MX を替えた瞬間に全員の入口が Google になる話から、片付けの順まで。

https://zenn.dev/pollenjp/articles/2026-10-30-split-delivery

こちらも、約 11 分の解説動画にした。

https://www.youtube.com/watch?v=Ak4VXcrc5zA

## 参考

- [RFC 7208 (SPF)](https://www.rfc-editor.org/rfc/rfc7208.html)
- [RFC 6376 (DKIM)](https://www.rfc-editor.org/rfc/rfc6376.html)
- [RFC 9989 (DMARC)](https://www.rfc-editor.org/rfc/rfc9989.html)
- Google Workspace 管理者ヘルプ: [SPF を設定する](https://knowledge.workspace.google.com/admin/security/set-up-spf?hl=ja)・[DKIM を設定する](https://knowledge.workspace.google.com/admin/security/set-up-dkim?hl=ja)・[DMARC を設定する](https://knowledge.workspace.google.com/admin/security/set-up-dmarc?hl=ja)
- Gmail ヘルプ: [メール送信者のガイドライン](https://support.google.com/mail/answer/81126?hl=ja)
