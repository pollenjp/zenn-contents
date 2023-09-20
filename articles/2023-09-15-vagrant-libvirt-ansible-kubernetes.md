---
title: "vagrant-libvirt と Ansible で自宅鯖prod環境に近い複数ノードKubernetes環境をローカルに作る"
emoji: "🚢"
type: "tech"
topics: ["vagrant", "libvirt", "ansible", "Kubernetes", "kvm"]
published: true
---

本記事は IaC (Infrastructure as Code) をベースに複数ノードの Kubernetes 環境をローカルPC内に構築することを目指します.

## きっかけ

学習も兼ねて実際に自宅鯖で k8s を動かしていると, ノードの追加・削除やバージョン・構成変更時等によく壊れます (個人談). 壊れると環境を戻すのが非常に面倒なので, いろんな実験を躊躇うケースがありました.
そこで手軽に試せる「簡易staging環境 (以降, 単にstagingと呼称)」のためのスクリプトを組むことにしました.

## 記事の内容

Vagrant, KVM, Ansible を用いて複数ノード・複数control-plane構成の Kubernetes 環境をstaging用に構築します.

staging用VMはVagrant経由で起動し, prodに寄せたアクセスができるよう設計しました.
prod は元々 Ansible で展開していたため, inventory ファイルを切り替えるだけで staging, prod を使い分けができるようにしています.

:::message
prodとは筆者の自宅鯖の本番環境 (簡易版) を意味する.
:::

### 構成図・環境

以下が prod と今回作る staging それぞれの構成図になります. どちらにもLAN内DNSサーバーを置き, ドメイン名で相互アクセスできるようにしています.

[![Image from Gyazo](https://i.gyazo.com/495b96abeda35f468bba4355e6c0cde4.jpg)](https://gyazo.com/495b96abeda35f468bba4355e6c0cde4)

Local PC: Windows 11 Pro (WSL)

| 名称 | 役割 | OS |
|:--|:--|:--|
| `vm-dns.vagrant.home` | DNS server | Ubuntu 22.04 |
| `vm01.vagrant.home` | k8s control-plane node | Ubuntu 22.04 |
| `vm02.vagrant.home` | k8s control-plane node | Ubuntu 22.04 |
| `vm03.vagrant.home` | k8s worker node | Ubuntu 22.04 |
| `vm04.vagrant.home` | k8s worker node | Ubuntu 22.04 |

#### 注意: 冗長性

本記事の構成は control-plane node を2つ用意していますが, ロードバランサーを用意していないため不完全です. 理想としてはロードバランサーが control-plane を監視しておき, 一方が落ちたら他方のIP・ドメインに変更するような設定が望ましです (たぶん).

- `apiserver-advertise-address`
  - 現在は `vm01.vagrant.home` と同じIPアドレスを直接指定してしまっています.
  - 本来はロードバランサーのIPアドレスを指定するようにすると良いです.
- `control-plane-endpoint`
  - 現在は `k8s-cp-endpoint.vagrant.home` ドメインを `vm01.vagrant.home` と同じIPアドレスを指すようにDNSサーバー側で固定しています.
  - これはロードバランサーを用意するまでの一時的な処置です.

### コード

<https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes>

上記のリポジトリに本記事用に対応したコードを載せています.
各アプリケーションを用意したあとに以下のコマンドで一発で構築できます.
※自分の手元では初回構築に1時間～1時間半ほどかかりました.

```sh
make debug-k8s-setup
```

or

```sh
go run ./tools/cmd setup-vagrant-k8s
```

:::message
このリポジトリは個人の自宅インフラのプロジェクトから一部を抜粋し, 記事用に書き換えたものです. make でも十分なのですが, prod では Makefile が肥大化したため, go でスクリプトを書いています. そのため `go run` でも実行可能です.
:::

## 基礎知識

今回は IaC (Infrastructure as Code) として全てすることも目的の一つであり, 以下のような領域で使い分けています. どれも有名なツールですので既知の人は読み飛ばしてください.

| Tool | 用途 | 設定ファイルフォーマット |
|:--|:--|:--|
| Vagrant | VMの作成・起動・停止・削除 | Ruby |
| Ansible | 各OSのセットアップ | YAML |
| Kubernetes | アプリケーションの展開 (本記事の範囲外) | YAML |

### Vagrant

[Vagrant](https://www.vagrantup.com/)はVMを管理することができるツールになります.
構成ファイル (`Vagrantfile`) に起動したいVMのスペックやOSを記述し, 裏で Virtualbox や KVM などの仮想化ソフトウェアの操作し, VMの作成・起動を行っています.

Vagrantは [Vagrant Cloud](https://app.vagrantup.com) に様々なOSのBoxが公開されており, セットアップ済みのVM環境を利用することができます.
`vagrant` ユーザー作成やssh等も予め設定されているため頻繁にVMを作成・削除する際に向いています.

### vagrant-libvirt

Vagrantの仮想化ソフトウェアとしてデフォルトでは VirtualBox が想定されていますが, [vagrant-libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt) というプラグインを用いることで KVM (libvirt) を利用することができます.
Vagrant Cloud でも libvirt 用の Box が用意されています.

今回は自分の好みや手元で VirtualBox と Hyper-V の共存が難しい等の理由もあって KVM を利用しています.

### Ansible

[Ansible](https://www.ansible.com/) はプッシュベース型の設定管理ツールです. サーバー側で実行させたい処理を予め YAML 形式で記述 (playbook) しておき, 実行時にはコントロールクライアントからSSH経由でコマンドを逐次実行してくれます.
Chef や Puppet といったプルベース型の設定管理ツールとは異なり, クライアント側にエージェントをインストールする必要が無くシンプルです.

inventory ファイルという設定ファイルに接続先のサーバー情報や変数情報を記述しておき, その情報を元に playbook を実行することができます. prod用, staging用として異なる inventory ファイルを用意し 切り替えることで, 同じ処理 (playbook) を異なるサーバー構成に対して簡単に実行することができます.

inventory ファイルは ini 形式と YAML 形式などのように静的に記述することもできますが, [dynamic inventory](https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html) の機能を使うことでPythonコードとして動的に生成することもできます.

### rye

今回は Python の Package Manager として [rye](https://github.com/mitsuhiko/rye) を利用しています.
ryeはパッケージ管理だけではなく, 任意のPythonバージョンのインストールも行ってくれるため, 最近の言語の潮流に乗っており便利です.
サンプルプロジェクトでは Ansible やその他のツールの依存関係解決のために利用しており, 特にryeである必要はありません.
ここでは詳しくない人でも `rye run <some command>` によって venv 環境化でコマンドを叩いている考えれば問題ないと思います.

## 解説

冒頭と同じ図を以下に持ってきました.

[![Image from Gyazo](https://i.gyazo.com/495b96abeda35f468bba4355e6c0cde4.jpg)](https://gyazo.com/495b96abeda35f468bba4355e6c0cde4)

全体の流れは以下のようになります.

- Vagrant で VM を起動
- DNSサーバーに各VMのIPアドレスを登録 (Ansible)
- Kubernetes のセットアップ (Ansible)
  - `kubeadm init` でクラスターを初期化
  - `kubeadm join` でクラスターに追加

### Vagrant で VM を起動

[今回のVagrantfile](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/main/Vagrantfile) では以下のようなスペック構成にしています.
左から順に `name`, `cpu`, `memory`, `box`, `provider`, `description` という順番で, ドメイン名アクセスと同じアクセスができるように `.vagrant.home` を後ろに追加しています.

```rb
VM_SPEC_ARR = [
  VmSpecData.new('vm-dns.vagrant.home', 2, 2048, VAGRANT_BOX, 'libvirt', 'dns node'),
  VmSpecData.new('vm01.vagrant.home', 4, 4096, VAGRANT_BOX, 'libvirt', 'cp01 node'),
  VmSpecData.new('vm02.vagrant.home', 2, 4096, VAGRANT_BOX, 'libvirt', 'cp02 node'),
  VmSpecData.new('vm03.vagrant.home', 2, 2048, VAGRANT_BOX, 'libvirt', 'worker01 node'),
  VmSpecData.new('vm04.vagrant.home', 2, 2048, VAGRANT_BOX, 'libvirt', 'worker02 node')
].freeze
```

#### vagrant up

このまま `vagrant up` で起動できればよいのですが, 自分の環境下では一気に立ち上げようとするとメモリ割当等に失敗するケースによく遭遇しました.
そんな時は一台ずつ起動してあげれば大丈夫なはずです (時間はかかりますが...).

[`Makefile`](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/Makefile#L105C1-L112) には以下のように記述しており, 起動時には `make vagrant-up` 等で起動するのが良いでしょう.

```Makefile
.PHONY: vagrant-up
vagrant-up:  ##
        vagrant box update
        vagrant up vm-dns.vagrant.home
        vagrant up vm01.vagrant.home
        vagrant up vm02.vagrant.home
        vagrant up vm03.vagrant.home
        vagrant up vm04.vagrant.home
```

#### vgrant ssh-config

Ansible で VM にアクセスする際にはsshの設定がそのまま使われるため `~/.ssh/config` を参照します.
しかし, 今回は先程起動したVMにアクセスしたいので `vgrant ssh-config` によって生成した ssh_config 設定を利用します.

環境変数[ANSIBLE_SSH_ARG](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-ssh_args) を利用して ssh のオプションを渡すことができるため, 以下のように実行することで見た目上, たとえば `vm01.vagrant.home` のドメインにアクセスしているように扱うことができます.

[Makefile](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/Makefile#L91-L98)

```Makefile
.PHONY: debug
debug:
	${MAKE} vagrant-up
	-vagrant ssh-config > inventory/vagrant.ssh_config
	ANSIBLE_SSH_ARGS='-F inventory/vagrant.ssh_config' \
		rye run ansible-playbook \
			-i inventory/vagrant.py \
			playbooks/debug.yml
```

### Ansible Inventory

今回の構成では `inventory/vagrant.py` として dynamic inventory の機能を利用しています. Vagrant で起動したマシンの IP アドレスを取得し動的に inventory を生成するためです.

試行錯誤しながら雑に書いているため見づらいかもしれませんが単に以下のような設定 (JSON) を吐き出しているだけです.

```sh
rye run python ./inventory/vagrant.py --list | jq
```

[フルログ](https://gist.github.com/pollenjp/82a445298a311d732b05ef563a309f55)

```json
{
  // 略
  "k8s_cp_master": {
    "hosts": [
      "vm01.vagrant.home"
    ],
    "children": []
  },
  "k8s_other_nodes": {
    "hosts": [
      "vm02.vagrant.home",
      "vm03.vagrant.home",
      "vm04.vagrant.home"
    ],
    "children": []
  },
  // 略
}

```

### DNSサーバーに各VMのIPアドレスを登録 (Ansible)

prodでは専用のDNSサーバーを建てて名前解決をするようなことが多いと思います. そのため, 手元でも専用のDNSを建てて本番同様に名前解決できる環境を作ります.

`playbooks/dns_server.yml` にそのための設定を書いています.

[playbooks/roles/dns_server/tasks/main.yml](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/main/playbooks/roles/dns_server/tasks/main.yml)

- DNSサーバー用のVMに BIND をインストール
- 設定ファイルを編集し起動
  - 変数としてドメインと各VMのIPアドレスとの対応などを与えておき, template として zone ファイルを形成しています.

    ```zone
    $TTL	1d
    @	IN	SOA	ns1 root.localhost. (
            202309100	; Serial (size:uint32) (YYYYMMDDX: date+1桁index)
                    60	; 1w Refresh
                    30	; 1d Retry
                  120	; 4w Expire
                    30	; 1d Negative Cache TTL
          )
    @	IN	NS	ns1

    {% for v4_conf in item.value.ipv4  %}
    {% for name, addr in v4_conf.addresses.items()  %}
    {{ name }}	IN	A	{{ v4_conf.network_component }}.{{ addr }}
    {% endfor %}
    {% endfor %}

    {% for v6_conf in item.value.ipv6  %}
    {% for name, addr in v6_conf.addresses.items()  %}
    {{ name }}	IN	AAAA	{{ v6_conf.network_component }}{{ addr }}
    {% endfor %}
    {% endfor %}

    {% for name, actual_name in item.value.cnames.items()  %}
    {{ name }}	IN CNAME	{{ actual_name }}
    {% endfor %}
    ```

    inventoryの一部

    ```json
    "network_configs": {
      "name_server": "192.168.121.214",
      "dns": {
        "acl": {
          "internal_network": [
            "localhost",
            "192.168.121.0/24"
          ]
        },
        "domains": {
          "vagrant.home": {
            "ipv4": [
              {
                "network_component": "192.168.121",
                "addresses": {
                  "vm02": 124,
                  "vm03": 142,
                  "vm04": 132,
                  "vm01": 29,
                  "vm-dns": 214,
                  "ns1": 214
                }
              }
            ],
            "ipv6": [],
            "cnames": {
              "k8s-cp-endpoint": "vm01"
            }
          }
        }
      }
    }
    ```

- 各VMのネームサーバーの向き先を `vm-dns.vagrant.home` のIPに向ける

### Kubernetes のセットアップ (Ansible)

Kubernetes の構築は [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) を利用して行います.

必要な操作が `kubeadm init` と `kubeadm join` で異なるので別の playbook に分けています.

[Makefile](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/Makefile#L73-L89)

```Makefile
.PHONY: debug-k8s-setup
debug-k8s-setup:  ## debug the playbook (vagrant)
#	略
	ANSIBLE_SSH_ARGS='-F inventory/vagrant.ssh_config' \
		${MAKE} run \
			INVENTORY_FILE="inventory/vagrant.py" \
			PLAYBOOK="playbooks/k8s-setup-control-plane.yml"
	ANSIBLE_SSH_ARGS='-F inventory/vagrant.ssh_config' \
		${MAKE} run \
			INVENTORY_FILE="inventory/vagrant.py" \
			PLAYBOOK="playbooks/k8s-setup-join-node.yml"
```

#### k8s-setup-control-plane.yml

- [playbooks/k8s-setup-control-plane.yml](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/main/playbooks/k8s-setup-control-plane.yml)
- [playbooks/roles/k8s-control-plane/tasks/main.yml](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/main/playbooks/roles/k8s-control-plane/tasks/main.yml)

inventory で `k8s_cp_master` グループに含めているものに対して処理していきます.

- install kubernetes
- kubeadm init
  - [kubeadm init](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/playbooks/roles/k8s-control-plane/tasks/main.yml#L49-L55)

    ```yaml
    ---
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: InitConfiguration
    localAPIEndpoint:
        advertiseAddress: "{{ k8s_apiserver_advertise_address }}"
        bindPort: 6443
    ---
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    networking:
        serviceSubnet: "10.96.0.0/16" # default
        # --pod-network-cidr=10.244.0.0/16 is required by flannel
        podSubnet: "10.244.0.0/16"
        dnsDomain: "cluster.local" # default
    controlPlaneEndpoint: "{{ k8s_cp_endpoint }}:6443"
    ```

  - [kubeadm_config.yaml](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/playbooks/roles/k8s-control-plane/tasks/main.yml#L49-L56)

    ```yaml
    - name: Initialize kubeadm
      ansible.builtin.command:
        cmd: >-
          kubeadm init
            --skip-token-print
            --config /tmp/kubeadm_config.yaml
    ```

- その他のインストール
  - [flannel](https://github.com/flannel-io/flannel)
    - deploy時の設定をGit管理する意味も含めてmanifestは `flannel-config.yml` に保存しています.
    - [playbook](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/playbooks/roles/k8s-control-plane/tasks/main.yml#L93-L108)
  - [helm](https://github.com/helm/helm)

今回は `vm01.vagrant.home` を最初の control-plane とするため, inventory で `k8s_cp_master` グループに指定しています.

```sh
rye run python ./inventory/vagrant.py --list | jq
```

```json
{
  // 略
  "k8s_cp_master": {
    "hosts": [
      "vm01.vagrant.home"
    ],
    "children": []
  },
  // 略
}
```

:::message
そういえば, 最近 [Kubernetes apt/yum repository が変更されましたね](https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/). 今回のものには反映させていますが, 古いままの人はご注意してください. IaCで管理していると一度設定したら各種ホームページを訪れる機会が減り, 気づきにくかったりします. 自分だけかな？
:::

#### k8s-setup-join-node.yml

[playbooks/k8s-setup-join-node.yml](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/main/playbooks/k8s-setup-join-node.yml)

基本的にはinventory で `k8s_other_nodes` グループに含めているものに対して処理していきますが, 一部 `k8s_cp_master` グループ上で諸々の情報を取得しています.

- k8s_other_nodes: install kubernetes
- k8s_cp_master: `vm01.vagrant.home` で kubeadm の token, 証明書, certificate-key を作成・取得
- k8s_other_nodes: token, 証明書, certificate-key を用いて `kubeadm join`

control-plane にするか否かは inventory で指定しています.

```sh
rye run python ./inventory/vagrant.py --list | jq
```

```json
{
  // 略
  "_meta": {
    "hostvars": {
      "vm-dns.vagrant.home": {},
      "vm01.vagrant.home": {},
      "vm02.vagrant.home": {
        "k8s_is_control_plane": true
      },
      "vm03.vagrant.home": {
        "k8s_is_control_plane": false
      },
      "vm04.vagrant.home": {
        "k8s_is_control_plane": false
      }
    }
  }
}

```

### 全て実行

以下のコマンドのいずれかでできます.

```sh
make debug-k8s-setup
```

or

```sh
go run ./tools/cmd setup-vagrant-k8s
```

### 停止・削除

普通の vagrant コマンドとほぼ同じですが, 以下のコマンドで一括で停止・削除できます.

[Makefile](https://github.com/pollenjp/sample-vagrant-libvirt-ansible-kubernetes/blob/ab14d17111e28c4bbd586b492812eaf920b442d2/Makefile#L114-L121)

停止だけ

```sh
make vagrant-halt
```

停止＋削除

```sh
make vagrant-destroy
# make clean でもよい
```

```Makefile
.PHONY: vagrant-halt
vagrant-halt:  ##
	vagrant halt

.PHONY: vagrant-destroy
vagrant-destroy:  ##
	-${MAKE} vagrant-halt
	vagrant destroy -f
```

## おわりに

今回, 自分としてはとりあえずで動くものを作れたので満足です.
同じく簡易staging環境を作りたいという方の参考になれば幸いです.
ただ, まだ以下の問題が残っているので余力があるときに追加・改善したいと思います.

- ロードバランサーの追加
- VMの数や名前等が変わったときに複数箇所を修正する必要があるので1つの設定を参照させる
  - ( 簡易staging なのでそこまでする必要はない気もする )

本記事の誤字脱字に関してはプルリクの方も受け付けております.
