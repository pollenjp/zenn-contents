---
title: "古いメールサーバーから Google Workspace へ 1 人ずつ移すあいだ、SPF・DKIM・DMARC はどうこじれるのか"
emoji: "📦"
type: "tech"
topics: ["dns", "spf", "dmarc", "googleworkspace", "gmail"]
published: true
---

## はじめに

[前の記事](https://zenn.dev/pollenjp/articles/2026-10-29-spf-dkim-dmarc)で、SPF・DKIM・DMARC がメールのどこを見ているのかを整理した。今回はその続きで、古いメールサーバーから Google Workspace へ 1 人ずつ移すあいだ (分割配信) に、メールがどう届き、どこでつまずくのかを、架空の会社で追ってみる。しくみの説明は前の記事に任せて、ここでは移行のあいだの話に絞る。同じ話を、約 11 分の解説動画にもした。

https://www.youtube.com/watch?v=Ak4VXcrc5zA

:::message
2026-10-30 時点の情報。画面の名前や手順は変わることがあるので、設定するときは Google のヘルプも見てほしい。会社と人は架空で、ドメインと IP アドレスは説明用。
:::

## TL;DR

- 分割配信は、MX を Google に向け、Google にアカウントがある人の分は Gmail に、無い人の分は旧サーバーへ中継する仕組み
- MX を替えた瞬間に、移っていない人の分も含めて、社外からのメールは全部 Google が入口になる
- 移行のあいだ、メールは 6 つの道を通る。SPF の `mx`・中継・旧サーバーの中の配達がハマりどころ
- DMARC は `p=none` のまま送り手を洗い出し、旧サーバーを片付けてから強める

## 架空の会社の状況

example.co.jp のメールは、共用のレンタルサーバー (以下、旧サーバー。legacy.example.net) にある。社員はメールソフトで旧サーバーから受け、旧サーバーから送っていて、MX も旧サーバーを指している。これを 1 年ほどかけて、1 人ずつ Google Workspace へ移したい。

全員を一度に移せば話は簡単だが、1 人ずつ移すと、そのあいだずっと「Google の人」と「旧サーバーの人」が同じ example.co.jp のアドレスで並ぶ。ここでは、移った A さん、まだ移っていない B さん、取引先 (example.com) の C さんで考える。

## 分割配信の設定は 2 つ

Google Workspace の分割配信 (split delivery) は、MX を Google に向け、届いたメールを宛先で振り分ける仕組み。Google にアカウントがある人の分は Gmail に、無い人の分は旧サーバーへ中継する。同じメールを両方に届ける二重配信とは別物。

![分割配信は宛先によって片方に届け、二重配信は同じメールを両方に届ける](/images/2026-10-30-split-delivery/02-split-dual.png)

管理コンソールでやるのは 2 つ。

1. [アプリ] → [Google Workspace] → [Gmail] → [ホスト] で、旧サーバーを経路として足す (TLS はできるだけ使う)
2. [ルーティング] で、[すべての無効なアカウントと認識できないアカウント] を旧サーバーへ回すルールを足す。当てるのは [受信メッセージ] と [内部受信メッセージ] の両方

つまり、Google にアカウントを作った人から順に、その人宛てのメールが Gmail に届くようになる。アカウントを先にまとめて作ると、まだ移っていない人のメールまで Gmail に入ってしまうので、移す人の分だけ作る。

厄介なのは、1 人で試すだけでも、MX を替えた瞬間に社外からのメールが全員分 Google を入口にするところ。Google 側が止まれば、移っていない人の受信も止まる。旧サーバーにつながらないと、Google は最長 7 日ほど送り直してから送り主へ返すとのこと。MX の TTL を先に短くして、すぐ戻せるようにしておく。

## 移行のあいだの 6 つの道

移行のあいだ、メールは次の 6 つの道を通る。

![移行のあいだの 6 つの道。社外から受けるメールは全部 Google が入口になる](/images/2026-10-30-split-delivery/03-six-paths.png)

- ① 社外 → A さん: Google が受けて、そのまま Gmail へ
- ② 社外 → B さん: Google が受けて、旧サーバーへ中継
- ③ A さん → B さん: Gmail から旧サーバーへ中継
- ④ B さん → A さん: 旧サーバーの中を通る (ここが曲者)
- ⑤ A さん → 社外: Google から
- ⑥ B さん → 社外: 旧サーバーから

①② は相手のドメインの SPF と署名で、③〜⑥ は自分のドメインの SPF で確かめられる。つまり自分の SPF には、Google と旧サーバーの両方が要る。

## ハマりどころ

- **SPF の `mx`**: 今の SPF が旧サーバーを `mx` で許していると、MX を替えた瞬間に `mx` が指すのは Google の受信サーバーになり、旧サーバーが名簿から落ちる (⑥ が通らない)。替える前に `v=spf1 a:legacy.example.net include:_spf.google.com ~all` のように、名前か IP で書いておく

![MX を替えると、SPF の mx が指す相手も変わる。旧サーバーを mx だけで許していると、名簿から落ちる](/images/2026-10-30-split-delivery/05-spf-mx-trap.png)

- **中継すると相手の SPF が合わない (②)**: 旧サーバーから見ると、メールを渡してくるのは Google なので、C さんのドメインの SPF は通らない。DKIM の署名は残るので、DMARC は DKIM で合格できる。旧サーバーが SPF で厳しく弾かないか、MX を替える前に旧サーバーの管理者と確かめて試しておく
- **社内宛ても中継に乗せる (③)**: Gmail から見ると B さんのアドレスは自分のドメインなので、ルールを [内部受信メッセージ] にも当てていないと届かない
- **旧サーバーは中で配る (④)**: B さんが A さんに送ると、旧サーバーは MX を見ずに、中にある A さんの旧い箱に入れてしまう。A さんはもう Gmail しか見ていないので気づかない。旧い箱は、Google のテスト用アドレス (`a@example.co.jp.test-google-a.com`) への転送専用にする。a@example.co.jp へ転送すると、また中で配られてループする

![旧サーバーは自分の担当のドメイン宛てを中で配る。移った人の旧い箱を、テスト用アドレスへの転送専用にする](/images/2026-10-30-split-delivery/06-loop.png)

- **DKIM は送るシステムごと**: セレクタで鍵を分けて、Google (`google`) と旧サーバーの鍵を並べる。旧サーバーが署名できないなら、⑥ の DMARC は SPF だけが頼り
- **DMARC は `p=none` のまま**: 問い合わせのフォームや複合機のような、忘れていた送り手がレポートで見えてくる

## 片付けは順番に

全員が移ったら、次の順に片付ける。

1. 旧サーバーから送るものが無くなったことを、DMARC のレポートで確かめる
2. 旧サーバーにしか無いアドレス (共有のアドレスや別名など) が残っていないか確かめる
3. SPF から旧サーバーを外す
4. 旧サーバーの DKIM の鍵を消す
5. 旧サーバーで example.co.jp のメールを止める
6. 分割配信のルールを外す
7. DMARC を `p=quarantine` へ、必要なら `p=reject` へ

送り手が残ったまま DMARC を強めると、そのメールが届かなくなるので、強めるのは最後。

## おわりに

分割配信の設定そのものは、管理コンソールで 2 つだけ。ややこしいのは、2 つのシステムが同じドメインで並ぶあいだに、SPF・DKIM・DMARC と旧サーバーの配達がどう絡むか。MX を替える前に、SPF の書き方と、旧サーバーが中継や社内宛てをどう扱うかを確かめておきましょうね。

参考になれば幸い。誤りがあればコメントで教えてほしい。

## 参考

- [前の記事: SPF・DKIM・DMARC がメールのどこを見ているのか](https://zenn.dev/pollenjp/articles/2026-10-29-spf-dkim-dmarc)・[前の動画 (約 13 分)](https://www.youtube.com/watch?v=07GTwfhbKsQ)
- Google Workspace 管理者ヘルプ: [分割配信を使用して 2 つのメールシステムにメールを送信する](https://knowledge.workspace.google.com/admin/gmail/advanced/send-email-to-2-email-systems-with-split-delivery?hl=ja)・[テスト用メールアドレス](https://knowledge.workspace.google.com/admin/gmail/test-email-addresses?hl=ja)
- Google Workspace 管理者ヘルプ: [SPF を設定する](https://knowledge.workspace.google.com/admin/security/set-up-spf?hl=ja)・[DKIM を設定する](https://knowledge.workspace.google.com/admin/security/set-up-dkim?hl=ja)・[DMARC を設定する](https://knowledge.workspace.google.com/admin/security/set-up-dmarc?hl=ja)
- [RFC 7208 (SPF)](https://www.rfc-editor.org/rfc/rfc7208.html) 5.4 節 (`mx`)・付録 D (転送と中継)
- [RFC 5321 (SMTP)](https://www.rfc-editor.org/rfc/rfc5321.html) 6.3 節 (ループ)
