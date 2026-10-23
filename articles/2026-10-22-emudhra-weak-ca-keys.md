---
title: "CA の秘密鍵が「4」だった — eMudhra の中間 CA の件を HTTPS の基礎から"
emoji: "🔑"
type: "tech"
topics: ["tls", "https", "security", "pki", "ecdsa"]
published: false
---

インターネットの「本物」を保証する認証局 (CA) の秘密鍵が、「4」でした。

たとえ話ではありません。2026 年 10 月、インドの認証局 eMudhra が 2025 年 4 月に作った中間 CA の証明書 8 枚で、秘密鍵がどれも 1 バイトに収まる整数 (4・56・66・115・155・156・230・237) だったことが分かりました。本来は 116 桁もある数の範囲から、乱数で選ぶはずの値です。誰でも一瞬で鍵を計算でき、その鍵があれば、どのドメインの証明書でも作れる状態でした。

主要なブラウザはすでに遮断しています。一方で、curl や Python、CI、IoT 機器のような「ブラウザ以外」は、手元の CA の束 (信頼するルートを並べたファイル) を更新するまで守られません。

この記事では、HTTPS の信頼の仕組みという基礎から始めて、何が起きたのか、なぜ一瞬で割れたのか、どう止められたのか、そして自分は何をすればよいのかを順に書きます。

![EM OV TLS CA - G3C の秘密鍵。48 バイトのうち、最後の 1 バイト以外がすべて 0 だった](/images/2026-10-22-emudhra-weak-ca-keys/key4.png)

:::message
2026-10-22 時点の情報です。話はまだ動いていて、eMudhra の正式な事故報告はまだ出ていません。日付と時刻は UTC で書きます (日本時間は 9 時間足してください)。
:::

同じ内容を、約 12 分の解説動画にもしました。

https://www.youtube.com/watch?v=r2F8RQnGTps

## HTTPS の「本物」は誰が保証しているか

ブラウザで `https://` のサイトを開くと、通信は暗号化されます。ただ、暗号化だけでは足りません。話している相手が本物の example.com ではなく、途中に割り込んだ偽者だったら、偽者と安全に話すことになってしまうからです。

そこで使うのが、公開鍵と証明書です。

- 鍵は 2 つで 1 組です。秘密鍵で署名し、公開鍵でその署名を確かめます。公開鍵は誰に配っても構いません
- 証明書は「この公開鍵は example.com のもの」と、認証局 (CA) が署名した文書です
- サーバは接続のたびに証明書を見せ、その公開鍵と組になる秘密鍵を持っていることを、署名で示します

ブラウザは、証明書に付いた CA の署名を、CA の公開鍵で確かめます。では、その CA の公開鍵が本物だとなぜ言えるのでしょうか。ここで「信頼の連鎖」が出てきます。

![サーバ証明書から署名をたどり、ルートストアにあるルートまで届けば信頼する](/images/2026-10-22-emudhra-weak-ca-keys/chain.png)

- ブラウザや OS は、信頼する「ルート CA」の一覧 (ルートストア) を持っています。どのルートを一覧に入れるかは、ブラウザや OS の運営者 (ルートプログラム) が審査して決めます
- ルートの鍵はふつうオフラインで大事にしまっておき、日々の発行は「中間 CA」が担います。中間 CA の証明書には、ルートが署名します
- サーバ証明書から上へ署名をたどり、一覧にあるルートまで届けば信頼します

この仕組みの急所は CA の秘密鍵です。中間 CA の秘密鍵を手にした人は、「この公開鍵は google.com のもの」という証明書を自分で作れます。ブラウザは署名をたどって一覧のルートに届くので、その証明書を本物として扱ってしまいます。

### 自分で見てみよう

この関係は、手元でも確かめられます。

**ブラウザで**: アドレス欄の鍵 (サイト情報) のアイコンから証明書を表示すると、階層 (ルート・中間・サーバ) が見えます。

**コマンドで**: `openssl s_client` で、サーバが送ってくる証明書の鎖を表示できます。`s:` がその証明書の名前、`i:` が発行者 (親) です。次は 2026-10-22 に手元で example.com を見た結果の一部です。

```console
$ openssl s_client -connect example.com:443 -showcerts
…
Certificate chain
 0 s:CN=example.com
   i:C=US, O=SSL Corporation, CN=Cloudflare TLS Issuing ECC CA 3
 1 s:C=US, O=SSL Corporation, CN=Cloudflare TLS Issuing ECC CA 3
   i:C=US, O=SSL Corporation, CN=SSL.com TLS Transit ECC CA R2
 2 s:C=US, O=SSL Corporation, CN=SSL.com TLS Transit ECC CA R2
   i:C=US, O=SSL Corporation, CN=SSL.com TLS ECC Root CA 2022
…
```

0 が example.com のサーバ証明書、1 と 2 が中間 CA で、2 の発行者 (SSL.com TLS ECC Root CA 2022) が、手元のルートストアにあるルートです。`i:` を上へたどると、ルートに着きます。

**ルートストアは**: OS やブラウザの設定から一覧を見られます。Windows は証明書マネージャ (certmgr.msc) の「信頼されたルート証明機関」、macOS はキーチェーンアクセスの「システムルート」、Linux は `/etc/ssl/certs` など、Firefox は設定の「証明書を表示」です。

## 何が起きたか

2025-04-30、eMudhra は鍵生成式典を開き、中間 CA を 24 枚作りました。

鍵生成式典は、CA の鍵の組を作る決まった手順です。業界の基準 (Baseline Requirements 6.1.1.1) は、手順書 (Key Generation Script) に従い、物理的に守られた場所で、複数の担当者のもと、HSM (鍵を外に出さずに保管・署名する専用の装置) のような暗号モジュールの中で鍵を作ることを求めています。ルートの鍵などでは、監査人が立ち会うか全工程を録画することも必須です。

24 枚は、A・B・C の 3 系列それぞれに、RSA と P-384 (楕円曲線を使う署名の方式の 1 つ) の 2 方式で、4 つの用途 (DV・OV・EV・コード署名) の分を作った形です。そのうち B・C 系列の P-384 の鍵を持つ 8 枚で、秘密鍵が 1 バイトに収まる整数でした。

| 中間 CA | 用途 | 秘密鍵 d |
| --- | --- | --- |
| EM DV TLS CA - G3B | TLS (サーバ証明書の発行) | 156 |
| EM DV TLS CA - G3C | TLS | 230 |
| EM EV TLS CA - G3B | TLS | 115 |
| EM EV TLS CA - G3C | TLS | 56 |
| EM OV TLS CA - G3B | TLS | 155 |
| EM OV TLS CA - G3C | TLS | 4 |
| EM OV CS CA - G3B | コード署名 | 66 |
| EM OV CS CA - G3C | コード署名 | 237 |

8 つの値は Scott Helme の記事によるものです。Mozilla も、8 つの鍵がどれも 256 未満の整数だったと書いています。

TLS 用の 6 枚は Chrome・Firefox・Apple・Microsoft のルートプログラムで信頼され、コード署名用の 2 枚は Microsoft が信頼していました。

信頼のつながり方には、特徴が 2 つあります。

![同じ 8 つの鍵に、親が 2 つつながっていた](/images/2026-10-22-emudhra-weak-ca-keys/parents.png)

- **親が 2 つあった**: 同じ鍵に、eMudhra のルート (emSign ECC Root CA - G3) から相互証明書を経てつながる証明書と、Sectigo のルート (AAA Certificate Services) が署名した「相互署名」(同じ鍵の証明書に、別のルートも署名したもの) の証明書がありました。片方の親を切っても、もう片方から信頼がつながります
- **Microsoft は途中の証明書もルートとして信頼していた**: 間にある emSign Root TLS CA - G3 と emSign Root CS CA - G3 (TLS 用とコード署名用) には、同じ名前の自己署名のルートもあり、Windows はそれらをルートとして直接信頼していました

eMudhra の説明では、8 枚のうち本番に使ったのは EM DV TLS CA - G3C の 1 枚だけで、8 枚のどれからも証明書は発行していません。TLS 用の 6 枚については、Chrome も、10-17 の遮断より前にはサーバ証明書が Certificate Transparency (CT、後述) のログに 1 枚も無かったと書いています。

## なぜ一瞬で割れたのか

P-384 の鍵は、楕円曲線の上の点を使って作ります。

- 曲線には、決まった点 G (ベースポイント) が定められています
- 秘密鍵 d は整数で、公開鍵 Q は「点 G を d 回足した点」です。Q = dG と書きます
- d から Q は一瞬で計算できますが、Q から d を求めるのはとても難しい。これが楕円曲線暗号の安全性の土台です

ただし、この「難しい」には前提があります。d が広い範囲から乱数で選ばれていることです。P-384 では、d を 1 から n − 1 までの範囲から選びます。n は約 3.9 × 10¹¹⁵、10 進で 116 桁の数です。正しく作った鍵を、知られている汎用の最良の方法 (Pollard の rho 法など) で解こうとすると、約 √n ≈ 6 × 10⁵⁷ 回の計算が要ります (手元の計算)。1 秒に 1 兆回計算できても、宇宙の年齢 (約 138 億年) の 10²⁸ 倍以上かかります。

今回の d は、どれも 255 以下でした。48 バイトのうち、最後の 1 バイト以外がすべて 0 です。116 桁の暗証番号のはずが、実は 3 桁以下だったようなものです。

![本来は約 3.9 × 10¹¹⁵ 通りから選ぶ数が、255 通りしかなかった](/images/2026-10-22-emudhra-weak-ca-keys/keyspace.png)

公開鍵 Q は証明書に載っています。あとは G、2G、3G … と順に計算して、Q と比べるだけです。

```
P = G; d = 1
while P != Q:            # Q は証明書に載っている公開鍵
    P = P + G; d = d + 1 # 楕円曲線上の点の足し算
# 最大 255 回で止まる。そのときの d が秘密鍵
```

たとえば d = 4 の鍵なら、G、2G、3G と進み、4 つ目の 4G で Q と一致します。どの鍵も最大 255 回で必ず止まります。手元で試すと、最適化していない素の Python でも、255 通りを全部試して約 0.04 秒でした。Scott Helme も、8 つの鍵すべてを 1 秒かからずに復元したと書いています。

厄介なのは、公開鍵の見た目からは弱さが分からないことです。Q は曲線上の正しい点で、形式の検査はすべて通ります。弱いかどうかは、実際に試してみるまで分かりません。

## 鍵が割れると何ができるか

中間 CA の秘密鍵があれば、好きなドメインの証明書を作れます。Mozilla も Cloudflare も、どのドメインの偽の証明書でも作れる状態だったと書いています。

ただし、偽の証明書だけでは足りません。TLS で相手になりすますには、利用者の通信を途中で横取りできる立場 (同じ Wi-Fi、乗っ取ったルータ、DNS の書き換えなど) も要ります。その立場があれば、本物のサイトのふりをして、通信を読んだり書き換えたりできます。

![偽の証明書と、通信を横取りできる立場がそろうと、なりすませる](/images/2026-10-22-emudhra-weak-ca-keys/forge.png)

ここで壊れたのは「相手が本物か」の確認 (認証) で、TLS の暗号そのものではありません。暗号は正しく働いたまま、偽者と安全に話してしまう、という壊れ方です。

コード署名は、ソフトに発行者の署名を付けて、誰が作ったか (と、改ざんされていないこと) を示す仕組みです。コード署名用の鍵には、別の怖さがあります。Windows はコード署名用のルートを直接信頼していたので、この中間 CA で署名したプログラムは、実在の発行者が署名したソフトに見えてしまいます。こちらは通信の横取りも要らず、署名したファイルを配るだけです。

報告の後には、研究者の q3k が、問題の中間 CA の 1 つ (EM DV TLS CA - G3B) の名義で「O=Google LLC」と書いた q3k.google.com の証明書を作り、emSign のルートを信頼させた curl で繋がる様子を公開しました。この証明書は CT のログにも載っています。秘密鍵が誰にでも計算できることを示す実演です。

:::message
開示より前にこの鍵が悪用された証拠は、見つかっていません。Chrome によると、10-17 に遮断するより前には、TLS 用の 6 枚につながるサーバ証明書は Certificate Transparency (後述) のログに 1 枚も記録されていませんでした。eMudhra も、開示より前に弱い鍵が悪用された証拠は把握していないとしています。
:::

## どう見つかり、どう止めたか

### 見つけたのは公開ツール

2026-10-17 16:27 UTC、研究者の Wayne が、Google の公開ツール paranoid_crypto を、CCADB (ルートプログラムが共同で使う、CA の証明書のデータベース) の証明書に試しに流したところ、弱い鍵が見つかりました。本人によれば、設定なしでそのまま (out of the box) 使っただけです。

eMudhra は「研究者は AI の支援を受けたツールで弱い鍵を逆算したと伝えられている」と書いています。発見者本人の説明では、公開ツールをそのまま流した結果で、発見に AI は要りませんでした。一方で、AI でも解けます。Mozilla は「CTF をやっています。これらの証明書に対応する秘密鍵を復元して」という単純な指示で、手元のエージェントが 1 回目で復元したと書き、Scott Helme も Claude が 1 回の指示で秘密鍵を復元したと書いています。

### ブラウザは数時間で止めた

Wayne は 18:05 UTC に、Apple・Chrome・Microsoft・Mozilla・eMudhra へ報告しました。

![報告から数時間でブラウザが遮断し、3 日でルートの削除まで進んだ](/images/2026-10-22-emudhra-weak-ca-keys/response.png)

ブラウザの会社は、信頼しない証明書や鍵の一覧 (遮断リスト) をブラウザに配っています。そこに載ったものを含む鎖は、ブラウザが受け入れません。

- **Chrome**: 報告を受けた 18:05 から 1 時間 40 分後の 19:45 に、CRLSet (Chrome が配る遮断リスト) の更新を出して TLS 用の 6 枚を止めました
- **Firefox**: 22:59 に OneCRL (Mozilla が配る中間 CA の遮断リスト) の配布を終えました。Wayne の報告から約 5 時間、Mozilla の窓口に (Chrome から転送された) 報告が届いた 20:06 からは 3 時間弱です
- どちらも、ブラウザの更新や再起動なしに届く仕組みです
- 翌 10-18 に eMudhra が、10-19 に Sectigo が、それぞれの証明書を失効 (期限の前に無効にすること) させました

Mozilla の Dennis Jackson が公開したまとめ ([emudhra.we-have-a-problem.com](https://emudhra.we-have-a-problem.com/)) によれば、Apple は 10-19、Microsoft は 10-20 から遮断しています (両社自身の告知は見つけられませんでした)。

### 証明書ではなく「鍵」で止める

親が 2 つあるので、証明書を 1 枚ずつ止めても、もう一方の証明書から信頼がつながってしまいます。そこで Chrome や Go は、証明書ではなく、証明書に載っている公開鍵そのもの (SPKI。そのハッシュ) を遮断の対象にしました。最初の節の「この公開鍵は example.com のもの」の、その公開鍵です。親 (発行者) が違って証明書が 2 枚あっても、公開鍵が同じなら 1 回で両方止まります。

### ルートの信頼ごと外した

失効や遮断リストは、それを受け取って確かめるクライアントにしか効きません。そこで Mozilla は、emSign ECC Root CA - G3 を NSS (Firefox などが使う暗号ライブラリ) のルートの一覧から外しました (NSS 3.131.1、10-20)。理由として、複数の人が独自に鍵を復元し、中には公表した人もいること、失効の情報を使わないクライアントも守る必要があることを挙げています。Chrome も、このルートを外した Chrome Root Store v43 を出し、10-27 (太平洋時間) に Stable に届く見込みです。

なぜルートを外すと、失効を見ないソフトまで守れるのでしょうか。失効を確かめないソフトも、鎖をたどって、いちばん上のルートが手元の CA の束 (信頼するルートを並べたファイル) にあるかは必ず確かめます。束からルートを消せば、そのルートに着く鎖は、失効を見なくても全部通らなくなります。ただし効くのは、手元の束 (OS の ca-certificates、NSS、言語のランタイムに同梱の束など) が更新されてからです。これが次の節の「CA の束を更新する」につながります。

![束からルートを消すと、失効を見ないソフトでも鎖が通らなくなる](/images/2026-10-22-emudhra-weak-ca-keys/bundle.png)

### Certificate Transparency が効いた

Certificate Transparency (CT) は、発行された証明書を、誰でも見られる追記専用のログに記録させる仕組みです。Chrome や Safari、Firefox は、ログに載っていない証明書を受け入れません。TLS 用の 6 枚 (「何が起きたか」の表の上の 6 枚) の下では、Chrome が遮断した 10-17 より前に、サーバ証明書が 1 枚もログに無かったので、CT を求めるブラウザには、偽の証明書が通る余地がありませんでした。

### 巻き添えは約 3 万枚

ルートの信頼を外すと、同じルートの下にある、侵害されていない中間 CA が出したサーバ証明書も使えなくなります。Chrome が CT のログで数えると、このルートにつながる期限内の証明書 (precertificate) は約 3 万 800 枚あり、その 99.9% が InCommon (米国の大学・研究機関向けの証明書サービス) の中間 CA が出したものでした。eMudhra も、約 3 万枚の移行が要ると見積もっています。

eMudhra は、この巻き添えについて 3 つのことを説明しています。

- **影響は、米国の 1 つの顧客 (とその利用者) に限られる**: eMudhra は顧客の名前を出していませんが、Chrome の集計でも、約 3 万枚のほぼ全部が InCommon の中間 CA の発行分でした
- **別の階層から出し直す**: その顧客の証明書は、ブラウザがまだ信頼している別のルートの下から出し直しています
- **G3 の下の発行を止めた**: 10-20 には、emSign ECC Root CA - G3 の下のすべての中間 CA からの発行を止めたと報告しました。外すルートの下では、新しい証明書をもう出しません

その後、10-22 に eMudhra は、世界向けの SSL/TLS 証明書の事業から自ら退くと発表しました ([発表](https://emudhra.com/en/press-release/emudhra-to-exit-global-ssl-tls-certificate-business))。顧客の証明書は提携先のものに無償で替え、失効を確かめる仕組みは差し替えが済むまで続け、根本原因の報告も出すとしています。

## まだ危ないのは誰か・何をすればよいか

更新されたブラウザは、遮断リストと CT で守られました。残るのはブラウザ以外です。

![守られたのはブラウザで、残っているのはブラウザ以外](/images/2026-10-22-emudhra-weak-ca-keys/at-risk.png)

curl・Python・Node.js・Go・Java・git、パッケージマネージャ、CI、IoT 機器などの多くは、ディスク上の CA の束 (信頼するルートを並べたファイル) を信頼し、既定では失効も CT も確かめません。たとえるなら、失効の回覧板を読まない家です。CA の束からルートが消えるまで、問題の中間 CA を信頼し続けます。Scott Helme は、手元のサーバに偽の api.stripe.com の証明書を置き (本物の Stripe には触れずに)、既定の設定の curl・Python・Node.js がそのまま繋がることを確かめています。

とくに気をつけたいのは、`curl | sh` 型のインストーラや自動更新、クラウドの鍵や署名鍵を持つ CI/CD、サーバどうしの API 呼び出しです。先のまとめによれば、macOS も OS としては遮断していますが、標準の curl・git・python3 などは `/etc/ssl/cert.pem` を使い、失効を確かめません。10-22 時点で、そのファイルはまだ更新されていません。

### すること

1. **OS の CA の束を更新する**: ふつうのシステム更新 (apt・dnf・pacman など) で入ります (下の表)
2. **言語のランタイムとライブラリを上げる**: ソフトによって見ている CA の束が違うので、使っているものごとに確かめます (下の表)
3. **コンテナを作り直す**: イメージの中の CA の束は、ベースイメージを更新して作り直すまで古いままです
4. **検証を無効にしてエラーを黙らせない**: 移行の途中で証明書のエラーが出ても、`-k` や `verify=False` で通さない
5. **確かめる**: 次の 2 つが**どちらも証明書のエラーになれば**、問題の中間 CA を信頼していません

```sh
curl https://dv-g3b-emudhra.we-have-a-problem.com/
curl https://dv-g3b-aaa-emudhra.we-have-a-problem.com/
```

1 つ目は eMudhra のルート側、2 つ目は Sectigo 側の経路です。両方が止まっている必要があるので、2 つともエラーになるのを確かめます。2 つ目は、古い束でもエラーになることがあります (古い束が、もともと Sectigo のルートを TLS 用に信頼していない場合)。2 つ目だけで安心しないでください。

手元で試すと、同じ curl でも、CA の束を curl.se の 2026-09-25 の版にすると 1 つ目に繋がってしまい、2026-10-20 の版にすると証明書のエラーになりました。

確かめるのは、実際にアプリが使う経路で行います。ブラウザや curl での確かめで大丈夫と出ても、アプリが別の検証の仕組みを使っていれば、そのアプリが守られている証拠にはなりません (Cloudflare)。

| 使っているもの | 見ている CA の束 | すること |
| --- | --- | --- |
| curl・git・Python の標準ライブラリ (urllib など) | OS の束 | OS の更新 |
| requests・boto (AWS の SDK) | certifi | certifi を 2026.10.19 以上に |
| pip | pip に同梱の束 | pip を上げる |
| Node.js | Node.js に同梱の束 | 22.23.4・24.22.1・26.12.1 以上に上げる |
| Java | OS に同梱の JDK は OS の束。単体の JDK は JDK の中 | OS の更新か、配布元の告知に従って JDK を上げる |
| Go | Linux は OS の束。macOS・Windows は OS の検証 | OS の更新。1.27.3・1.26.10 で作り直すと、問題の鍵を拒むリストも入る |

Go のプログラムは標準ライブラリを中に含む (静的リンク) ので、拒むリストを効かせるにはビルドし直します (告知の言葉は「すぐに上げてほしい」)。Linux なら OS の束の更新だけでも、古いバイナリが問題のルートを信頼しなくなります。

ディストリの対応は次のとおりです (2026-10-22 時点。使っているディストリの告知も確かめてください)。

| ディストリ | 対応 |
| --- | --- |
| Ubuntu | USN-8944-1 (22.04・24.04・26.04・26.10)、USN-8944-2 (14.04〜20.04。Ubuntu Pro が要る) |
| Debian | ca-certificates 20250419+deb13u1 (trixie、DSA-6572-1)・20250419~deb12u2 (bookworm、DLA-4845-1) |
| Fedora | ca-certificates 2026.2.92_v9.0.319 |
| RHEL | 告知 RHSB-2026-012 あり。修正版はまだで、当面の手順を案内している |
| SUSE | SLES 12 SP5・11 SP4 に修正版 (SUSE-SU-2026:4821-1・4820-1) |
| Arch Linux | ca-certificates-mozilla 3.131.1-1 |
| Alpine | ca-certificates 20261020-r0 |
| Gentoo | ca-certificates 20260816.3.131.1 |
| FreeBSD | FreeBSD-SA-26:73 |
| NixOS | NixOS のモジュールが、問題のルートを既定で外すようになった (nixos-unstable と 26.05 は 10-20)。channel か flake の nixpkgs を上げて `nixos-rebuild switch` する。cacert 3.131.1 は 10-22 時点で nixos-unstable-small まで |

このモジュールの対策は `/etc/ssl/certs` に効くものなので、NixOS の外 (nix-darwin や Docker のイメージなど) で `pkgs.cacert` を使っている所には、別の手当てが要ります。

ディストリの更新が届くまでの手動の手順 (ルートを不信にする) も [we-have-a-problem のページ](https://emudhra.we-have-a-problem.com/) にありますが、Linux の手順には、ふつうの利用者には勧めないと書かれています。まずは公式の更新を入れるのが確実です。

更新できない機器 (古いルータやテレビなど) は、Mozilla 系の CA の束を持つものなら、ルートからの経路の途中にある相互証明書の期限、2039 年 7 月まで信頼し続けます。

:::message
Google は、emSign ECC Root CA - G3 につながる証明書を使っているサイトに、別の階層の証明書へすぐに取り替えるよう呼びかけています。Mozilla はこのルートを NSS から外し、Chrome も v43 で外す見込みです。
:::

## 前兆と原因

### eMudhra の説明: 鍵の組が 2 つできた

eMudhra は Bugzilla と告知で、原因の見立てをおおむね次のように説明しています。

1. FIPS 140-2 Level 3 に適合した HSM (鍵を外に出さずに保管・署名する専用の装置) の中で、まず本来の強い鍵の組を作った
2. 鍵を外へ持ち出す予定があったので、鍵を包んで出し入れする操作 (wrap/unwrap) を試す手順を、承認を得て式典の手順に加えた
3. その途中、コードの欠陥で、HSM の中に弱い鍵の組がもう 1 組できた
4. 2 組のラベルが似ていたので、誤って弱いほうの組で証明書の発行依頼 (CSR) を作り、自社のルートで署名した。同じ CSR を Sectigo にも送った
5. だから証明書には、弱いほうの公開鍵が載った

ほかに、次のことも書いています。

- RSA の中間 CA は別の経路で作ったので影響はない
- 工程の複数の段階で人為的なミスがあった。今後 1 週間で、これまでのすべての鍵生成式典を検証し直す
- 基盤の侵害やデータの漏えいは無い
- 正確な根本原因はまだ確定しておらず、コードの調査を続けている。確定したら公表する

eMudhra は、この問題が約 18 か月、2 回の独立した監査を経ても見つからなかったとも書いています (式典から発見までは約 17 か月半です)。

### 2025 年 6 月の CRL: 見逃された前兆

実は、この食い違いには前兆がありました。2025-06-05、Sectigo の Rob Stradling が、eMudhra の TLS 用の中間 CA 6 枚の CRL (失効した証明書の一覧) の署名が検証できない、と報告しています (Bug 1970728)。当時の eMudhra の説明は「手作業で CRL を作ったときに、違う鍵の設定を使った。正しい鍵で作り直した」というものでした。

![証明書と CRL の鍵の食い違いを、弱い鍵に合わせる形で解消していた](/images/2026-10-22-emudhra-weak-ca-keys/crl.png)

上の説明に照らすと、最初の CRL は本来の強い鍵で署名したので、弱い公開鍵の載った証明書と合わなかった。そこで弱いほうの鍵で署名し直した、とされています。「証明書の公開鍵と、CA が使っている鍵が違う」という兆候は、弱い鍵に合わせる形で消えていたことになります。

### ほかの人の推測

業界のメーリングリストや Bugzilla では、別の見方も出ています。どれも推測で、確かめられてはいません。

- HSM と連携するライブラリには、HSM の中の本物の鍵を指す「参照用の偽の秘密鍵」を使うものがある。その参照の値を、本物の秘密鍵として使ってしまったのではないか (Corey Bonnell、Matthew Hardeman)
- 鍵を HSM の中で作ったのか、外で作って取り込んだのかが問われている。確かめる手段として、HSM の中で作られたことを製造元の鍵で証明させる (attestation) ことを挙げる人もいる (Thor Simon)
- 承認された HSM の中の鍵生成だけでは、256 未満の秘密鍵にはならないはずだ。外から与えた値か、外の鍵の材料から作った鍵ではないか、という指摘 (Jacob Durham、10-22)

### まだ分かっていないこと

- d が 256 未満になった正確な仕組み
- 弱い鍵が HSM の中で作られたのか (eMudhra は中で作ったと説明している)
- 17 か月余りの間に、CT を確かめないクライアントが狙われたかどうか
- eMudhra の正式な事故報告 (CCADB の規則では、開示から 14 日以内に完全な報告を出す)

## まとめ

この件から見えるのは、Web の信頼が 4 つのものに支えられていることです。

![Web の信頼を支える 4 つ](/images/2026-10-22-emudhra-weak-ca-keys/pillars.png)

1. **乱数で作る鍵**: 公開鍵の見た目では弱さが分からないので、作る手順と、作った後の確かめが要になる
2. **透明性 (CT)**: 誰が何を発行したかが公開されているから、CT を求めるブラウザには偽の証明書が通らなかったと言える
3. **ルートプログラムの素早い取り消し**: 報告から数時間で遮断し、数日でルートの信頼まで外した
4. **更新されるクライアント**: どんな対策も、更新したクライアントにしか届かない

1 つ目が崩れても、ブラウザでは残りの仕組みが働き、開示より前に悪用された証拠は見つかっていません。ただし、CT を確かめないクライアントが狙われたかどうかは分かりません。ブラウザの外で 4 つ目を担うのは、私たち一人ひとりです。手元の CA の束、言語のランタイム、コンテナを、いま一度更新しておきましょう。

## 付録

### 用語集

| 用語 | 意味 |
| --- | --- |
| CA (認証局) | 「この公開鍵はこのドメインのもの」と署名して、証明書を出す組織 |
| DV・OV・EV | サーバ証明書の種類。ドメインの持ち主か (DV)、組織の実在まで (OV)、より厳しく (EV) 確かめて出す |
| ルート CA・中間 CA | ブラウザや OS が直接信頼する CA と、ルートの署名を受けて日々の発行を担う CA |
| ルートストア・ルートプログラム | 信頼するルートの一覧と、それを審査して決めるブラウザ・OS の運営者 |
| 相互署名・相互証明書 (cross-sign) | 同じ名前・同じ鍵の CA に、別の親が署名した証明書。信頼の経路が増える |
| 自己署名のルート | 自分の鍵で自分に署名した証明書。ルートストアに入ることで信頼される |
| ECDSA・P-384 | 楕円曲線を使う署名の方式と、そこで使う曲線の 1 つ |
| 秘密鍵 d・公開鍵 Q | Q = dG。G は曲線の上の決まった点 (ベースポイント) |
| SPKI | 証明書の中の公開鍵の部分。そのハッシュで「鍵そのもの」を指せる |
| 失効・CRL | 期限の前に証明書を無効にすること。CRL は CA が出す失効の一覧 |
| 遮断リスト (CRLSet・OneCRL) | ブラウザの会社が配る、信頼しない証明書や鍵の一覧。Chrome は CRLSet、Mozilla は OneCRL。ブラウザの更新なしに届く |
| CT (Certificate Transparency) | 発行された証明書を、誰でも見られる追記専用のログに記録させる仕組み |
| precertificate | CT のログに先に記録する、発行前の証明書の写し |
| HSM | 鍵を外に出さずに保管・署名する専用の装置 |
| 鍵生成式典 | CA の鍵の組を作る決まった手順。手順書に従い、HSM のような暗号モジュールの中で作り、監査人の立ち会いか録画で記録を残す |
| コード署名 | ソフトに発行者の署名を付けて、誰が作ったか (と改ざんされていないこと) を示す仕組み |
| CCADB | CA の証明書と情報を集めた、ルートプログラムが共同で使うデータベース |

### 時系列

| UTC | 日本時間 | 出来事 |
| --- | --- | --- |
| 2025-04-30 | 2025-04-30〜05-01 | 鍵生成式典で中間 CA を作る (8 枚の鍵が 1 バイト) |
| 2025-05-14 | 2025-05-14 | Sectigo の相互署名 (有効期間の始まり) |
| 2025-06-05 20:05 | 2025-06-06 05:05 | CRL の署名が検証できないと報告 (Bug 1970728) |
| 2025-06-06 07:30 | 2025-06-06 16:30 | 署名の通る CRL を公開し直す (のちに「弱い鍵で署名し直した」と説明) |
| 2026-10-17 16:27 | 10-18 01:27 | Wayne が paranoid_crypto で見つける |
| 10-17 18:05 | 10-18 03:05 | Apple・Chrome・Microsoft・Mozilla・eMudhra に報告 |
| 10-17 19:45 | 10-18 04:45 | Chrome が CRLSet で遮断 |
| 10-17 20:06 | 10-18 05:06 | Mozilla の窓口に、Chrome から転送された報告が届く |
| 10-17 22:59 | 10-18 07:59 | Firefox の OneCRL の配布が終わる |
| 10-18 09:24〜09:38 | 10-18 18:24〜18:38 | eMudhra が 8 枚を失効 |
| 10-19 14:55 | 10-19 23:55 | Sectigo が相互署名の 8 枚の失効を報告 |
| 10-20 02:13 | 10-20 11:13 | Go 1.27.3 / 1.26.10 |
| 10-20 12:15 | 10-20 21:15 | q3k が、q3k.google.com の証明書の実演を公開 |
| 10-20 12:54 | 10-20 21:54 | NSS 3.131.1 (emSign ECC Root CA - G3 を削除) |
| 10-20 13:01 | 10-20 22:01 | Mozilla が公表 |
| 10-20 14:14 | 10-20 23:14 | Chrome Root Program が公表 |
| 10-20 16:27 | 10-21 01:27 | eMudhra が、G3 の下の発行を止めたと報告 |
| 10-22 | 10-22 | eMudhra が、世界向けの SSL/TLS 証明書の事業から退くと発表 |
| 10-27 朝 (太平洋時間) | 10-27 夜〜10-28 未明 | Chrome Root Store v43 が Stable に届く見込み |

### 出典

- Mozilla Bugzilla [Bug 2083449](https://bugzilla.mozilla.org/show_bug.cgi?id=2083449) (今回の件)・[Bug 1970728](https://bugzilla.mozilla.org/show_bug.cgi?id=1970728) (2025 年の CRL)
- Mozilla Security Blog: [Protecting Firefox users following the compromise of eMudhra intermediate certificates](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/)
- mozilla.dev-security-policy の[スレッド](https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/0R3XLJL6FuE) (Mozilla・Chrome Root Program の公表、Wayne の説明、推測の投稿)
- Google: [Protecting Chrome and Android users from compromised eMudhra intermediate CA certificates](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/)
- Cloudflare: [eMudhra certificate compromise](https://blog.cloudflare.com/emudhra-certificate-compromise/)
- Scott Helme: [Eight Bits of Trust: eMudhra's CA keys cracked in milliseconds](https://scotthelme.co.uk/eight-bits-of-trust-emudhras-ca-keys-cracked-in-milliseconds/)
- eMudhra: [告知](https://emudhra.com/en/post/certificate-security-incident)・[プレスリリース](https://emudhra.com/en/press-release/emudhra-statement-distrust-global-tls-intermediate-root-ca)・[SSL/TLS の事業から退く発表 (10-22)](https://emudhra.com/en/press-release/emudhra-to-exit-global-ssl-tls-certificate-business)
- [emudhra.we-have-a-problem.com](https://emudhra.we-have-a-problem.com/) (対象の証明書・各ソフトとディストリの対応・確かめ方)
- Go: [golang-announce](https://groups.google.com/g/golang-announce/c/66pojueaQek)、Node.js: [v22.23.4](https://github.com/nodejs/node/releases/tag/v22.23.4)、OpenJDK: [Vulnerability Advisory 2026-10-21](https://openjdk.org/groups/vulnerability/advisories/2026-10-21)、certifi: [GHSA-6q48-6c7w-9gw2](https://github.com/advisories/GHSA-6q48-6c7w-9gw2)、curl: [CA certificates extracted from Mozilla](https://curl.se/docs/caextract.html)
- Ubuntu: [USN-8944-1](https://ubuntu.com/security/notices/USN-8944-1)・[USN-8944-2](https://ubuntu.com/security/notices/USN-8944-2)、Debian: [DSA-6572-1](https://security-tracker.debian.org/tracker/DSA-6572-1)、Red Hat: [RHSB-2026-012](https://access.redhat.com/security/vulnerabilities/RHSB-2026-012)、SUSE: [SUSE-SU-2026:4821-1](https://www.suse.com/support/update/announcement/2026/suse-su-20264821-1/)、FreeBSD: [FreeBSD-SA-26:73](https://www.freebsd.org/security/advisories/FreeBSD-SA-26:73.caroot.asc)、NixOS: [nixpkgs#576485](https://github.com/NixOS/nixpkgs/issues/576485)
- q3k の[投稿](https://social.hackerspace.pl/@q3k/117473185371937796)、Google の [paranoid_crypto](https://github.com/google/paranoid_crypto)
- 鍵生成式典の決まり: [Baseline Requirements 6.1.1.1](https://github.com/cabforum/servercert/blob/main/docs/BR.md#6111-ca-key-pair-generation)
- P-384 の定数: [NIST SP 800-186](https://csrc.nist.gov/pubs/sp/800/186/final)、鍵の作り方: [FIPS 186-5](https://csrc.nist.gov/pubs/fips/186-5/final)、解くのに要る計算の目安: [Handbook of Applied Cryptography 3 章](https://cacr.uwaterloo.ca/hac/about/chap3.pdf)・[NIST SP 800-57 Part 1](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- Go の静的リンク: [Go FAQ](https://go.dev/doc/faq#Why_is_my_trivial_program_such_a_large_binary)、事故報告の期限: [CCADB Incident Reporting Guidelines](https://www.ccadb.org/cas/incident-report)、24 枚の開示: [Bug 1965559](https://bugzilla.mozilla.org/show_bug.cgi?id=1965559)
- 国内の報道: [ITmedia NEWS (2026-10-21)](https://www.itmedia.co.jp/news/article/2610/21/2000002444/)
