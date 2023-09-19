---
title: "vagrant-libvirt と Ansible で自宅鯖prod環境に近い複数ノードKubernetes環境をローカルに作る"
emoji: "🚢"
type: "tech"
topics: ["vagrant", "libvirt", "ansible", "Kubernetes"]
published: false
---

本記事は IaC (Infrastructure as Code) をベースに複数ノードの Kubernetes 環境をローカルPC内に構築することを目指します.

## きっかけ

学習も兼ねて実際に自宅鯖で k8s を動かしていると, ノードの追加・削除やバージョン・構成変更時等によく壊れます (個人談).
壊れると環境を戻すのが非常に面倒なので, いろんな実験を躊躇うケースがありました.

そこで手軽に試せる「簡易staging環境 (以降, 単にstagingと呼称)」を手軽に構築したいと考えていました.

## 記事の内容

IaC (Infrastructure as Code) をベースに
複数ノードの Kubernetes 環境をstaging用に構築します.

staging用マシンはVagrant経由で起動し, prodに寄せたアクセスができるよう設計しました.
prod は元々 Ansible で展開していたため, inventory ファイルを切り替えるだけで staging, prod を使い分けができるようにしています.

### 構成図

以下が今回作る prod, staging それぞれの構成図になります.
どちらにもLAN内DNSサーバーを置き, ドメイン名で相互アクセスできるしています.

[![Image from Gyazo](https://i.gyazo.com/495b96abeda35f468bba4355e6c0cde4.jpg)](https://gyazo.com/495b96abeda35f468bba4355e6c0cde4)

- ホストマシン: Windows 11 Pro (WSL)
  - KVM (Kernel-based Virtual Machine)
    - DNS server (BIND) x 1
      - `vm-dns.vagrant.home`
    - k8s control-plane node x 2
      - `vm01.vagrant.home`
        - ※ `apiserver-advertise-address`
        - ※ `k8s-cp-endpoint.vagrant.home` (`control-plane-endpoint`)
      - `vm02.vagrant.home`
    - k8s worker node x 2
      - `vm03.vagrant.home`
      - `vm04.vagrant.home`

### 注意

本記事の構成は control-plane node を2つ用意していますが, ロードバランサーを用意していないため不完全です.
理想としてはロードバランサーが control-plane を監視しておき, 一方が落ちたら他方のIP・ドメインに変更するような設定が望まし気がします.

- `apiserver-advertise-address`
  - 現在は `vm01.vagrant.home` と同じIPアドレスを直接指定してしまっています.
  - 本来はバランサーのIPアドレスを指定するようにすると良いと思います.
- `control-plane-endpoint`
  - 現在は `k8s-cp-endpoint.vagrant.home` ドメインを `vm01.vagrant.home` と同じIPアドレスを指すようにDNSサーバー側で固定しています.
  - これはロードバランサーを用意するまでの一時的な処置です.

### コード

<https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes>

上記のリポジトリに本記事用に対応したコードを置いています.

```sh
make debug-k8s-setup
```

or

```sh
go run ./tools/cmd setup-vagrant-k8s
```

:::message

このリポジトリは個人の自宅インフラのプロジェクトから一部を抜粋し, 記事用に書き換えたものです.
make command でも十分なのですが, 自宅インフラプロジェクトでは Makefile が肥大化したため, go でスクリプトを書いています. 好みに応じて `go run` でも実行可能です.

:::

## 基礎知識

今回は IaC (Infrastructure as Code) として全てすることも目的の一つであり, 以下のような領域で使い分けています.

| Tool | 用途 | 設定ファイルフォーマット |
|:--|:--|:--|
| Vagrant | VMの作成・起動・停止・削除 | Ruby |
| Ansible | 各ノードのセットアップ | YAML |
| Kubernetes | アプリケーションの展開 (本記事の範囲外) | YAML |

どれも有名なツールですので既知の人は読み飛ばしてください.

### Vagrant

VagrantはVMを管理することができるツールになります.
構成ファイル (`Vagrantfile`) に起動したいVMのスペックやOSを記述し,
裏で Virtualbox や KVM などの仮想化ソフトウェアの操作し, VMの作成・起動を行っています.

Vagrantは [Vagrant Cloud](https://app.vagrantup.com) に様々なOSのBoxが公開されており,
セットアップ済みのVM環境を利用することができます.
`vagrant` ユーザー作成やssh等も予め設定されているため
頻繁にVMを作成・削除する際に向いています.

### vagrant-libvirt

Vagrantの仮想化ソフトウェアとしてデフォルトでは VirtualBox が想定されていますが,
[vagrant-libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt) というプラグインを用いることで KVM (libvirt) を利用することができます.
Vagrant Cloud でも libvirt 用の Box が用意されています.

今回は自分の好みや手元で VirtualBox と Hyper-V の共存が難しい等の理由もあって KVM を利用しています.

### Ansible

Ansible はプッシュベース型の設定管理ツールです. サーバー側で実行させたい処理を予め YAML 形式で記述 (playbook) しておき, 実行時にはコントロールクライアントからSSH経由でコマンドを逐次実行してくれます.
Chef や Puppet といったプルベース型の設定管理ツールとは異なり, クライアント側にエージェントをインストールする必要が無くシンプルです.

inventory ファイルという設定ファイルに接続先のサーバー情報や変数情報を記述しておき, その情報を元に playbook を実行することができます. prod用, staging用として異なる inventory ファイルを用意し 切り替えることで, 同じ処理 (playbook) を異なるサーバー構成に対して簡単に実行することができます.

inventory ファイルは ini 形式と YAML 形式などのように静的に記述することもできますが, [dynamic inventory](https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html) の機能を使うことでPythonコードとして動的に生成することもできます.

今回の構成では `inventory/vagrant.py` として dynamic inventory の機能を利用しています. Vagrant で起動したマシンの IP アドレスを自動取得し, 動的に inventory を生成するためです.
