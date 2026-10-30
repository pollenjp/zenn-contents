---
title: "CVE-2026-98432 について調べてみた — 11 年潜んでいた Linux の TCP の脆弱性と2行の修正" # 記事のタイトル
emoji: "📦" # アイキャッチの絵文字 (1 文字)
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "kernel", "tcp", "security"] # タグ。["git", "nix"] のように書く
published: true # true にして to-zenn.sh で zenn-contents に写すと、merge で公開される
---

:::message
2026-10-30 時点の情報。Ubuntu と RHEL はまだ修正版を出していない途中なので、このあと状況は変わるかも。
:::

## はじめに

X で CVE-2026-98432 に関する情報が話題になっていたので調べてみることにした。Linux カーネルの脆弱性で、直したのは `kfree()` を `kfree_rcu()` に替える 2 行。原因の変更は 2015 年なので、11 年近く潜んでいたことになる。一方で、深刻さの点数は初め 9.8 から 5.9 まで割れていて、10-30 には Red Hat が 5.9 を 8.1 (暫定) に上げていた。

TCP の接続の受け付けと RCU のおさらいも兼ねて、何が起きているのかを、シーケンス図を中心にざっくりまとめる。どの処理とどの処理が重なると壊れるかまでを書き、攻撃の手順は書かない。同じ話を、約 15 分の解説動画にもした。

https://www.youtube.com/watch?v=JXba-BX3tzM

## TL;DR

- IPv4 の TCP が接続を受け付ける途中で、IP オプションの写し (以下、箱) を解放したあとも、別の CPU が読み続けてしまう use-after-free
- 原因は 2015 年の変更 (Linux 4.4 から)。直し方は、解放を `kfree()` から `kfree_rcu()` に替える 2 行
- 評価は初め、カーネルの CNA (CVE の番号を振る組織) が 9.8、Ubuntu が 8.3、Red Hat が 5.9 と割れた。10-30 には Red Hat が 8.1 (暫定) に上げていた。10-27 には Anthropic が、multipath TCP で待ち受けるサーバで root を取る攻撃コードを Claude が書いたと公表している
- 上流の保守中の系列はすべて直った。Ubuntu と RHEL はまだ (10-30 時点)。**修正入りのカーネルに上げて再起動**が本命

## 2 つの CPU が、同じ request socket を触る

TCP の受け付けでは、SYN を受けたサーバがまず request socket (受け付け途中の控え) を作り、ACK が届いたら本物の子ソケットを作る。アプリが `accept()` で受け取るのは子ソケットのほう。

ここで、最初の SYNACK が途中で遅れると、相手は待ちきれずに SYN を送り直し、遅れて届いた SYNACK にも ACK を返す。つまりサーバには、再送の SYN と ACK がほぼ同時に届く。SYNACK がまったく届かなければ ACK は返ってこないので、重なるのはこういう遅れて届いた場合など。

![最初の SYNACK が遅れると、再送の SYN と ACK がほぼ同時に届く](/images/2026-10-29-linux-tcp-ip-options-uaf/delay.png)

Linux 4.3 までは、どちらのパケットもリスナーのロックを取ってから処理していたので、同じ request socket を 2 つの CPU が同時に触ることはなかった。2015 年の変更 (commit 079096f103fa、Linux 4.4 から) で、request socket は確立した接続と同じ表 (ehash) に入り、ロックなしで直接見つかるようになった。

![request socket は、Linux 4.4 から確立した接続と同じ表 (ehash) に入り、ロックを取らずに直接見つかる](/images/2026-10-29-linux-tcp-ip-options-uaf/ehash.png)

速くなった代わりに、2 つの処理が同じ request socket の上で重なる。

![Linux 4.4 からは、ACK と再送の SYN がロックなしで同じ request socket の上で重なる](/images/2026-10-29-linux-tcp-ip-options-uaf/lock-4-4.png)

## 箱が先に捨てられる

SYN に IP オプション (通ったルータを記録する Record Route など。ふだんの通信ではまず使われない) が付いていると、Linux は返事の SYNACK にも載せるため、その写しを request socket に置く。この写しが今回の主役で、ここでは箱と呼ぶ。

![① SYN のときに箱ができる。④ で箱が解放されたあと、⑤ で CPU B が解放済みの箱を読む](/images/2026-10-29-linux-tcp-ip-options-uaf/race.png)

再送の SYN を処理する CPU B は、RCU の読み取り区間の中で箱の場所を読み (②)、SYNACK を組み立て始める。同じころ、ACK を処理する CPU A は子ソケットを作り、箱をコピーせずにそのまま渡す (③)。接続が切れるなどして子ソケットがすぐに壊されると、箱は `kfree()` ですぐに解放される (④)。CPU B はそれを知らずに、解放済みの箱の中身で SYNACK を組み立ててしまう (⑤)。

持ち主が新しい部屋に移ったあとも、受付の係が控えた棚の番号で荷物を使い続け、その間に棚には別の荷物が置かれている、みたいな感じ。

しかも CPU B は、箱の中の長さを 2 回読む。間で中身が入れ替わると、パケットの先頭に空けた長さと、書く長さが食い違う。カーネルの CNA は、範囲の外を書き換えたり、カーネルが止まったりしうると見ている。

## 直し方は 2 行

RCU は、読む側と解放する側の約束。読む側は読み取り区間の中で読み、解放する側は、その区間にいる CPU が全員抜けるまで待ってから解放する (`kfree_rcu()`)。`kfree()` はその場ですぐに解放するので、片方だけ守っても効かない。

今回、箱を読む側はもともと RCU で読んでいて、ほかの解放の経路も `kfree_rcu()` だった。外れていたのは、IPv4 で子ソケットと request socket を壊すときの 2 か所の `kfree()` だけで、修正 (commit e3f33b0a1d89) はその 2 か所を `kfree_rcu()` に替えた。

![左が直す前、右が直した後。kfree_rcu() は、CPU B が読み取り区間を抜けるまで解放を待つ](/images/2026-10-29-linux-tcp-ip-options-uaf/fix.png)

2 か所目の `tcp_v4_reqsk_destructor()` は、request socket を別のリスナーへ移す機能 (既定は無効) の経路を塞ぐ。修正は最初 1 行で、netdev のメーリングリストの AI のレビューの bot がこの経路を指摘し、2 行になったとのこと。

IPv6 は、子ソケットに渡すときに箱の写しを作るので、1 つ目の経路の影響を受けない。ただし IPv6 のソケットでも、IPv4 の接続は IPv4 の処理を通る。

## 評価は 9.8 から 5.9 まで割れ、Red Hat が 8.1 に上げた

3 者とも「ネットワーク越しに、権限も操作も要らない」というところは同じで、割れたのは、起こしやすさと、起きうることの見立て。

- カーネルの CNA (9.8、CVSS 3.1): 外れても何度でも試せるので、起こすのは易しい。漏えい・書き込み・停止のどれもありうる
- Ubuntu (8.3、CVSS 4.0): 競合という前提があることを AT:P で書き分け、漏えいと改ざんは小さく見た
- Red Hat (初めは 5.9、CVSS 3.1): 競合に勝つのは難しく、現実的な最悪は停止

ところが 10-30 に確かめ直したら、Red Hat は 8.1 (まだ暫定) に上げていた。競合に勝つのは難しい (AC:H) という見立てはそのままで、漏えいと改ざんも大きいと見直している。これでカーネルの CNA とは、起きうること (C・I・A) の見立てがそろい、違いは起こしやすさ (AC) だけになった。Ubuntu は、漏えいと改ざんを小さく見たまま。

10-27 には、Anthropic の Frontier Red Team が記事で、multipath TCP で待ち受ける Linux サーバに、離れた所から root としてコマンドを動かす攻撃コードを Claude が書いたと公表した。報告者の Xinyang Ge も、記事の寄稿者に名を連ねている。一方で、公開された攻撃コードと、実際に悪用されたという報告は見当たらず、CISA の KEV (10-28 版) にも載っていない。

各組織の深刻さの点数だけでなく、ベクタと、誰がどの前提で付けたかを読むと、自分の環境に当てはめやすくなる。

## 何をすればよいか

本命は、修正入りのカーネルに上げて、再起動すること。上流は保守中の 8 系列 (5.10〜7.3) のすべてに修正が入った。Debian は修正済みで、Ubuntu は 10-29 の更新の時点で修正版がまだ無い。RHEL 8・9・10 は、初めは修正を先送り (Fix deferred) にしていたけど、10-30 には「影響あり (Affected)」に変わっていた。近いうちに修正を出すかもしれない、という扱いで、修正版はまだ無い (10-30 時点)。

```sh
uname -r                          # 動いているカーネルの版
sysctl net.ipv4.tcp_migrate_req   # 0 で、移し替えの BPF も無ければ 2 つ目の経路は閉じている
ss -Mln                           # multipath TCP で待ち受けているソケット
```

※ Go 1.24 以降で作ったサーバは、既定で multipath TCP で待ち受けることがある。ただし、MPTCP の待ち受けが無ければ安全、というわけではなく、穴そのものは IPv4 の TCP の受け付けにある。Go のセキュリティチームも 10-29 に、`GODEBUG=multipathtcp=0` で既定を外せるとしつつ、それはもろいつなぎで、本命はカーネルの更新だと告知している。

修正版を待つ間は、届く相手を絞ることと、移し替えを無効のままにすることがつなぎになる。IP オプション付きの TCP を入口で捨てる nftables の規則も考えられるが、公式の回避策ではなく、効き目も試していない (動画で触れている)。

## おわりに

ポインタを別の持ち主に渡すなら、まだ読んでいる処理をどう守るかを決めておく必要がある。コピーして渡すか (IPv6 はそうしていた)、RCU で守るなら解放も待つか。今回は、そのうち 2 か所だけが約束から外れていて、それが 11 年近く残っていた。

カーネルは、パッケージを入れ替えただけでは動いているものが変わらない。手元のサーバで動いているカーネルの版を、いま一度確かめておきましょうね。

もし誤っているところがありましたらごめんなさい。コメントで教えてもらえると助かる。

## 参考

- 修正の commit e3f33b0a1d89: https://git.kernel.org/linus/e3f33b0a1d89d738e2b913dab519226eaf76e15b
- 原因の commit 079096f103fa: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=079096f103faca2dd87342cca6f23d4b34da8871
- カーネルの CNA の告知: https://git.kernel.org/pub/scm/linux/security/vulns.git/plain/cve/published/2026/CVE-2026-98432.mbox
- Ubuntu: https://ubuntu.com/security/CVE-2026-98432
- Debian: https://security-tracker.debian.org/tracker/CVE-2026-98432
- Red Hat: https://access.redhat.com/security/cve/CVE-2026-98432
- Anthropic「Discovering vulnerabilities with Claude」(2026-10-27): https://www.anthropic.com/research/discovering-vulnerabilities-with-claude
- Go 1.24 のリリースノート (net の MPTCP の既定): https://go.dev/doc/go1.24
- Go のセキュリティチームの告知 (2026-10-29。oss-security への転送): https://www.openwall.com/lists/oss-security/2026/10/30/1
