---
title: "CA の秘密鍵が「4」だった件をざっくり追ってみた (eMudhra)"
emoji: "🔑"
type: "tech"
topics: ["tls", "https", "security", "pki", "ecdsa"]
published: true
---

## はじめに

最初に聞いたとき、インターネットの「本物」を保証する認証局の秘密鍵が「4」と言われて、何のことやらと思った。どうやら 2026 年 10 月、インドの認証局 eMudhra が 2025 年 4 月に作った中間 CA の証明書 8 枚で、秘密鍵がどれも 1 バイトに収まる整数だったとか。

これを機会におさらいしようと思い、自分用のメモも兼ねてざっくりまとめる。HTTPS の基礎から順に追いたい人向けに、約 12 分の解説動画も作った。

https://www.youtube.com/watch?v=r2F8RQnGTps

:::message
2026-10-22 時点の情報。eMudhra はまだ原因を調べている途中で、詳しい報告書は出ていないので、細かいところは変わるかも。時刻は UTC (日本時間は +9 時間)。
:::

## TL;DR

- eMudhra の中間 CA 8 枚で、P-384 の秘密鍵が 4・56・66… と、どれも 255 以下だった
- 証明書に載っている公開鍵から順に試すだけで鍵が分かり、どのドメインの証明書でも作れる状態だった
- Chrome と Firefox は報告から数時間で遮断した (Apple・Microsoft も数日で続いたらしい)。開示より前に悪用された証拠も見つかっていない
- **curl・Python・CI・コンテナのような「ブラウザ以外」は、手元の CA の束を更新するまで守られない**

## 何が起きたか

2025-04-30、eMudhra は鍵生成式典を開いて中間 CA を 24 枚作った。そのうち B・C 系列の P-384 (楕円曲線を使う署名の方式の 1 つ) の鍵を持つ 8 枚で、秘密鍵 d が 4・56・66・115・155・156・230・237 だったらしい (値は Scott Helme さんの記事から。Mozilla も、8 つとも 256 未満だったと書いている)。

![EM OV TLS CA - G3C の秘密鍵。48 バイトのうち、最後の 1 バイト以外がすべて 0 だった](/images/2026-10-22-emudhra-weak-ca-keys/key4.png)

8 枚のうち 6 枚は TLS 用で、Chrome・Firefox・Apple・Microsoft が信頼していた。残りの 2 枚はコード署名用で、Microsoft が信頼していた。

## なぜ一瞬で割れるのか

P-384 の秘密鍵 d は、1 から n − 1 までの範囲から乱数で選ぶ。n は約 3.9 × 10¹¹⁵、10 進で 116 桁もある数。公開鍵 Q は、曲線の上の決まった点 G を d 回足した点 (Q = dG) で、d から Q は一瞬で出せても、Q から d を求めるのはとても難しい。

今回の d はどれも 255 以下。つまり 48 バイトのうち、最後の 1 バイト以外は全部 0。116 桁の暗証番号のはずが、実は 3 桁以下だった、みたいな感じ。

公開鍵 Q は証明書に載っているので、あとは G、2G、3G … と順に足して Q と比べるだけでいい。

```text
P = G; d = 1
while P != Q:            # Q は証明書に載っている公開鍵
    P = P + G; d = d + 1 # 楕円曲線上の点の足し算
# 最大 255 回で止まる。そのときの d が秘密鍵
```

手元の素の Python でも、255 通りを全部試して約 0.04 秒だった。厄介なのは、公開鍵の見た目は曲線の上の正しい点で、形式の検査では弱さに気づけないところ。

## 鍵が割れると何ができるか

中間 CA の秘密鍵があれば、好きなドメインの証明書を作れる。ブラウザは署名をたどって信頼済みのルートに着くので、その証明書を本物として扱ってしまう。

とはいえ、偽の証明書だけではなりすませない。同じ Wi-Fi・乗っ取ったルータ・DNS の書き換えのような、通信を横取りできる立場も要る。ここで壊れたのは「相手が本物か」の確認 (認証) で、TLS の暗号そのものではない。暗号は正しく働いたまま、偽者と安全に話してしまう類の壊れ方になる。

コード署名用の 2 枚のほうは、通信の横取りすら要らない。Windows はコード署名用のルートを直接信頼していたので、この中間 CA で署名したプログラムは、実在の発行者のソフトに見えてしまう。

## 見つかってから止まるまで

見つけたのは研究者の Wayne さん。Google の公開ツール [paranoid_crypto](https://github.com/google/paranoid_crypto) を、CCADB (ルートプログラムが共同で使う CA の証明書のデータベース) の証明書に設定なしでそのまま流したら引っかかったらしい。本人の説明では AI は要らなかったとのこと。とはいえ AI でも解けて、Mozilla は手元のエージェントに「CTF をやっています」と頼んだら 1 回目で秘密鍵を復元したと書いている。

![報告から数時間でブラウザが遮断し、3 日でルートの削除まで進んだ](/images/2026-10-22-emudhra-weak-ca-keys/response.png)

- 10-17 18:05 に Apple・Chrome・Microsoft・Mozilla・eMudhra へ報告
- Chrome は 1 時間 40 分後の 19:45 に、遮断リスト (CRLSet) の更新を出した
- Firefox は 22:59 に、遮断リスト (OneCRL) の配布を終えた (報告から約 5 時間)
- 10-18 に eMudhra が、10-19 に Sectigo が証明書を失効させた
- 10-20 に Mozilla はルート (emSign ECC Root CA - G3) ごと NSS から外した。Chrome も Root Store v43 で外す見込み

個人的に面白かったのは、同じ鍵に親が 2 つあったところ。eMudhra のルートにつながる証明書のほかに、Sectigo のルートが同じ鍵に相互署名した証明書もあって、証明書を 1 枚ずつ止めても、もう片方から信頼がつながってしまう。なので Chrome や Go は、証明書ではなく公開鍵そのもの (SPKI) を遮断の対象にしていた。

Certificate Transparency (CT) も効いていて、TLS 用の 6 枚の下では、遮断より前のサーバ証明書が CT のログに 1 枚も無かった。CT を求めるブラウザには、偽の証明書が通る余地が無かったことになる。

ルートごと外したので、侵害されていない中間 CA の下の証明書 (約 3 万枚) も巻き添えで移行が要る。eMudhra は 10-22 に、世界向けの SSL/TLS 証明書の事業から退くと発表した。

## ブラウザ以外はまだ危ない

更新されたブラウザは、遮断リストと CT で守られ、残るのはブラウザ以外。curl・Python・Node.js・Go・Java・git、パッケージマネージャ、CI、IoT 機器などの多くは、ディスク上の CA の束 (信頼するルートを並べたファイル) を信頼し、既定では失効も CT も確かめない。失効の回覧板を読まない家のようなもの。

![守られたのはブラウザで、残っているのはブラウザ以外](/images/2026-10-22-emudhra-weak-ca-keys/at-risk.png)

なので、手元の CA の束から問題のルートが消えるまでは、偽の証明書がそのまま通ってしまう。やることはこのあたり。

- OS の CA の束を更新する (apt・dnf・pacman などの普段のシステム更新で入る)
- 言語のランタイム側の束も上げる (certifi は 2026.10.19 以上、Node.js は 22.23.4・24.22.1・26.12.1 以上、Go は 1.27.3・1.26.10 で作り直す)
- コンテナはベースイメージを更新して作り直す
- 移行の途中で証明書のエラーが出ても、`-k` や `verify=False` で黙らせない

確かめるときは、次の 2 つが**どちらも証明書のエラーになれば** OK (1 つ目は eMudhra のルート側、2 つ目は Sectigo 側の経路)。

```sh
curl https://dv-g3b-emudhra.we-have-a-problem.com/
curl https://dv-g3b-aaa-emudhra.we-have-a-problem.com/
```

自分の手元でも、curl に渡す CA の束を curl.se の 2026-09-25 の版にすると 1 つ目に繋がってしまい、2026-10-20 の版にするとちゃんとエラーになった。

※ 2 つ目は古い束でもエラーになることがあるので、2 つ目だけで安心しない。macOS の標準の curl・git・python3 が使う `/etc/ssl/cert.pem` は、10-22 時点でまだ更新されていない。ディストリごとの対応は [emudhra.we-have-a-problem.com](https://emudhra.we-have-a-problem.com/) にまとまっている。

## 原因はまだ調査中

eMudhra の説明では、HSM (鍵を外に出さずに保管・署名する専用の装置) の中に本来の強い鍵の組を作ったあと、鍵の出し入れを試す手順の途中で、コードの欠陥から弱い鍵の組がもう 1 組できたらしい。2 組のラベルが似ていたので、誤って弱いほうで証明書の発行依頼 (CSR) を作ってしまった、とのこと。ただ根本原因はまだ確定していなくて、「HSM の中で作るだけなら、256 未満の鍵にはならないのでは」という指摘も出ている。

ちなみに 2025 年 6 月には、この中間 CA の CRL (失効した証明書の一覧) の署名が検証できないという[報告](https://bugzilla.mozilla.org/show_bug.cgi?id=1970728)が出ていた。当時は「違う鍵の設定を使った」として署名し直していて、今の説明に照らすと、弱いほうの鍵に合わせて食い違いを消していたことになる。前兆はあったわけだ。

## おわりに

今回の件を追っていて、Web の信頼は 4 つに支えられているんだなと思った。乱数で作る鍵、透明性 (CT)、ルートプログラムの素早い取り消し、それと更新されるクライアント。

![Web の信頼を支える 4 つ](/images/2026-10-22-emudhra-weak-ca-keys/pillars.png)

1 つ目が崩れても、ブラウザでは残りの仕組みが働き、開示より前に悪用された証拠は見つかっていない。ただし、CT を確かめないクライアントが狙われたかどうかは分からない。ブラウザの外で 4 つ目を担うのは、我々一人ひとり。手元の CA の束、言語のランタイム、コンテナを、いま一度更新しておきましょうね。

## 参考

- [Mozilla Bugzilla Bug 2083449](https://bugzilla.mozilla.org/show_bug.cgi?id=2083449)
- [Protecting Firefox users following the compromise of eMudhra intermediate certificates (Mozilla Security Blog)](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/)
- [Protecting Chrome and Android users from compromised eMudhra intermediate CA certificates (Google)](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/)
- [mozilla.dev-security-policy のスレッド](https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/0R3XLJL6FuE) (Wayne さんの説明・原因の推測)
- [eMudhra certificate compromise (Cloudflare)](https://blog.cloudflare.com/emudhra-certificate-compromise/)
- [Eight Bits of Trust: eMudhra's CA keys cracked in milliseconds (Scott Helme)](https://scotthelme.co.uk/eight-bits-of-trust-emudhras-ca-keys-cracked-in-milliseconds/)
- eMudhra の[告知](https://emudhra.com/en/post/certificate-security-incident)・[SSL/TLS の事業から退く発表](https://emudhra.com/en/press-release/emudhra-to-exit-global-ssl-tls-certificate-business)
- [emudhra.we-have-a-problem.com](https://emudhra.we-have-a-problem.com/) (対象の証明書・各ソフトとディストリの対応・確かめ方)
