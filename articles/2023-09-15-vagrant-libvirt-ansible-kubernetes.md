---
title: "vagrant-libvirt と Ansible で自宅鯖prod環境に近い複数ノードKubernetes環境をローカルに作る"
emoji: "🚢"
type: "tech"
topics: ["vagrant", "vagrant-libvirt", "ansible", "Kubernetes"]
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

<!-- TODO: 構成図 -->

- ホストマシン: Windows 11 Pro (WSL)
  - KVM (Kernel-based Virtual Machine)
    - DNS server (BIND) x 1
      - `vm-dns.vagrant.home`
    - k8s control-plane node x 2
      - `vm01.vagrant.home`
      - `vm02.vagrant.home`
    - k8s worker node x 2
      - `vm03.vagrant.home`
      - `vm04.vagrant.home`

### 注意

本記事の構成は control-plane node を2つに冗長化していますが, まだ以下を実装していないため不完全です.

- `apiserver-advertise-address` をロードバランサーのIPに変更
  - 現在は `vm01.vagrant.home` のIPアドレスを直接指定しています.
- `api-server-endpoint` をロードバランサーのドメイン名に変更
  - 現在は `k8s-cp-endpoint.vagrant.home` ドメインを `vm01.vagrant.home` と同じIPアドレスを指すようにDNSサーバー側で固定しています.
  - これは後に容易に設定変更できるようにするための一時的な処置です.

## 基礎知識

今回は IaC (Infrastructure as Code) として全てすることも目的の一つであり, 以下のような領域で使い分けています.

| Tool | 用途 | 設定ファイルフォーマット |
|:--|:--|:--|
| Vagrant | VMの作成・起動・停止・削除 | Ruby |
| Ansible | 各ノードのセットアップ | YAML |
| Kubernetes | アプリケーションの展開 (本記事の範囲外) | YAML |

### Vagrant

VagrantはVMを操作することができるツールになります.
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

<!-- TODO: ansibleの説明 -->
