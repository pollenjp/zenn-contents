---
title: "ansible-lint のカスタムルールを利用して Ansible 内での変数命名規則を縛ってみた話"
emoji: "🚢"
type: "tech"
topics: ["ansiblelint", "ansible"]
published: true
---

## まえがき

みなさんこんにちは! KMCID: pollenjp です.
本記事は [KMC Advent Calendar 2023](https://adventar.org/calendars/8840) の 3 日目の記事です.

昨日の記事は taisei さんによる [Jsoo でスイカゲームもどき | A watermelon game on browser implemented in ocaml](https://taiseikmc.github.io/watermelon-game-jsoo/blog/index.html) でした.
元のスイカゲームは動画などでしか見たことはなかったのですが, **スイカゲームもどき**を触ってみて絶妙なゲームバランスに納得し楽しめました.

## はじめに

さて, 本記事では ansible-lint のカスタムルールを利用して Ansible 内での変数命名規則を縛ってみた話をします.

自分用に作った命名規則を [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) パッケージに落とし込んで利用しているので, もし同じ Ansible の悩みを抱えている方がいたら参考にしてみてください.

## Ansible の変数にスコープが無い

Ansible では様々な場面で変数の定義または代入が行われます. 例えば,

- inventory ファイルの `vars` セクションで定義して事前に渡す変数
- `ansible.builtin.set_fact` で定義した変数または書き換えた変数
- task の `register` で定義された変数

等です. しかしながら, これらによって定義される変数にはスコープが無く, 外部のタスクやロールに閉じていません. そのため, 任意の場所からアクセスができ値の書き換えが可能になっています. これらによる問題点をいくつか挙げてみます.

※ Ansible の role や tasks は見た目上, 関数呼び出しのように扱うことはできますが, 実際は外部に定義している task 配列を現在の playbook に取り込んでいるに過ぎません. そのため, スコープが無いのは当然かもしれません.

### 問題点 1: チーム内で変数の定義・利用場所の把握が難しい

現状, Ansible の [Language Server](https://github.com/ansible/ansible-language-server) では**変数の定義ジャンプ**等はまだサポートされていないため, 変数がどこで定義されたのかが非常にわかりにくくなっています.

一人だけで管理するプロジェクトであればまだしも, チームで管理するプロジェクトにおいて, ルール無く利用された変数を把握することは非常に困難でありバグの温床にもなります.

### 問題点 2: 変数を予期せず上書きしてしまうケース

実際にこのパターンは頻繁に踏むわけではありませんが, 例えば次のようなケースが考えられます.

以下のような inventory, role, playbook file 構成で `ansible-playbook` を実行した場合, 変数 `var__overwrite` がどのように変化するのかを追ってみます.

```sh
ansible-playbook -i inventory/debug.yml playbooks/debug.yml
```

```txt
zsh ❯ tree --charset=ascii
.
|-- ansible.cfg
|-- inventory
|   `-- debug.yml               ... (1) var__overwrite の初期値を定義
`-- playbooks
    |-- debug.yml               ... (2) var__overwrite の値を表示
    `-- roles
        `-- sample
            `-- tasks
                `-- sample.yml  ... (3) var__overwrite を上書き
```

`inventory/debug.yml`

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local
      var__overwrite: "original" # (1) var__overwrite の初期値を定義
```

`playbooks/roles/sample/tasks/sample.yml`

```yaml
---
- name: Overwrite parent playbook vars
  ansible.builtin.set_fact:
    # (3) var__overwrite を上書き
    var__overwrite: overwritten (by <some_role>/tasks/sample.yml)
```

`playbooks/debug.yml`

```yaml
---
- name: Overwrite Vars Play
  hosts:
    - localhost
  tasks:
    - name: Playbook 外で定義した変数の値を表示
      ansible.builtin.debug:
        msg: |
          var__overwrite: {{ var__overwrite }}
        ###############################
        # output                      #
        # (2) var__overwrite の値を表示 #
        ###############################
        # var__overwrite: original
    - name: Include role (中で var__overwrite を変数として再代入)
      ansible.builtin.include_role:
        name: sample
        tasks_from: sample.yml
    - name: 変数の値を表示
      ansible.builtin.debug:
        msg: |
          var__overwrite: {{ var__overwrite }}
        ###############################
        # output                      #
        # (2) var__overwrite の値を表示 #
        ###############################
        # var__overwrite: overwritten (by <some_role>/tasks/sample.yml)
```

上記の `playbooks/debug.yml` のコメントに記載していますが, `var__overwrite` の値は `ansible.builtin.include_role` で呼び出した role 内で上書きされています.
これが意図した動作であれば問題ありませんが, もしかすると `sample` role は role 内だけで利用する変数を定義したかっただけかもしれません. そのような場合, 値が予期せず変更され, そのまま次の task が実行されてしまいます.

## 命名規則で改善

上述した問題を完全に解決することは現状難しいですが, 変数側の命名規則を設けることで低コストで改善することは可能です. 例えば,

- 定数なのか変数なのか
- どこで定義・利用されているか
- 期待される変数スコープは何か

などの情報を変数名に含め lint で検知することで最悪のケースを回避する一助となります.

## 命名規則

ここでは [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) で定義している命名規則を紹介します.

### role 名, tasks 名を prefix に含める

まず, 「問題点 2: 変数を予期せず上書きしてしまうケース」 については, 変数名の prefix に変数のスコープに role や tasks 名を含めることで改善します.

そしてこの考えは ansible-lint の [`var-naming[no-role-prefix]`](https://ansible.readthedocs.io/projects/lint/rules/var-naming/) ルールに**一部**含まれています.

> var-naming[no-role-prefix]: Variables names from within roles should use `role_name_` as a prefix. Underlines are accepted before the prefix.

例えば `ansible.builtin.include_role` 等で `sample` という role を呼び出し, `vars` で変数を与える際に `sample_` という prefix を使いなさいということです. `sample` role が**受け取る変数**は `sample_` という prefix を付けることを示しています.

```yaml
- name: Run sample
  ansible.builtin.include_role:
    name: sample
  vars:
    sample_var1: "var1"
    sample_var2: "var2"
```

しかし, このルールはかなり弱めに設定されています. role が **受け取る変数** (playbook 側から渡す変数) にしか制約が働かないからです.
「問題点 2」で発生した問題は role 内で `ansible.builtin.set_fact` を使って変数を上書きしてしまうことでしたが, このルールでは明示的に変数を渡しているわけではないため検知でません.

また, `ansible.builtin.include_role` だけでなく `ansible.builtin.include_tasks` も外部の tasks を取り込んでいるため同様のルールを適応したいところです.

そこで, [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) では次のようなルールを設けています.

- **role の中で定義・代入する変数にはすべて `<role_name>_role__` prefix を付与する**
- **tasks の中で定義・代入する変数にはすべて `<tasks_name>_tasks__` prefix を付与する**

このルールは [`var-naming[no-role-prefix]`](https://ansible.readthedocs.io/projects/lint/rules/var-naming/) ルールを完全に内包しています.
加えて, `ansible.builtin.include_tasks` や role 内での `ansible.builtin.set_fact` 等にも適応されるため, 予期せず変数を上書きしてしまうケースを検知することができます.

[lint ルールの実装コード](https://github.com/pollenjp/ansible-lint-custom-strict-naming/blob/main/src/rules/var_name_prefix.py)

`roles/sample/tasks/main.yml` の例

```yaml
- name: Set fact
  ansible.builtin.set_fact:
    # role の中で定義・代入する変数にはすべて <role_name>_role__ prefix を付与する
    sample_role__var1: "var1"
    sample_role__var2: "var2"
```

### Playbook 内で定義され, 動的に値が変わりうるものに `var__` prefix を付与する

次に, 「問題点 1: チーム内で変数の定義・利用場所の把握が難しい」にも通ずるのですが,
Ansible の変数を扱う際にそれが **動的に値が変わる変数**として扱われているのか, **定数**として扱われているのか
が明示されていないと, 作業効率が落ちます. 変数を利用する際に値が期待したものであり, 途中で書き換わっていないかを判定する必要があるからです.

ここで inventory ファイルなどで事前に渡しておく変数は後々変更を加えるべきではないためこの変数のことを **定数** (const)と, Playbook 内などで `ansible.builtin.set_fact` により定義・代入される変数のことを **変数** (var)と記述することにします.

これらの 定数 (const) と 変数 (var) は分けようと意識していても名前衝突の可能性はまだ残っています. そのため, これらの変数にはそれぞれ prefix として `const__`, `var__` を付与することにしましょう.

現在の [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) では定数 (const) として検知しようとするとルールが複雑化するため `const__` prefix は検知していません.
そのため, 定数 (const) として変数を新たに作る場合には `var__` prefix をつけないように人間が気をつける必要があります.

※ 定数 (const) として扱いたいものに `var` や double underscore を繋げたような prefix をつける人はいないでしょうという気持ちから変数 (var) 側の prefix を `var__` にしています.

### ルールの結合

上記に上げた 2 つのルールは組み合わせて利用することができます. (ただし, 変数名が長くなるデメリットもある.)

|                 | 定数 (const) ※1            | 変数 (var)              |
| :-------------- | :------------------------- | :---------------------- |
| playbook 内     | `const__xxx`               | `var__xxx`              |
| sample role 内  | `sample_role__const__xxx`  | `sample_role__var__xxx` |
| sample tasks 内 | `sample_tasks__const__xxx` | `sample_role__var__xxx` |

※1 [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) では未検知

## ansible-lint のカスタムルールを作る

ansible-lint は [Custom linting rules - Ansible Lint Documentation](https://ansible.readthedocs.io/projects/lint/custom-rules/) にある通り, 独自の lint ルールを作ることができます.

プロジェクト特有のルールをローカルに作成する場合は設定ファイルで `rulesdir` を指定することでカスタムルールを利用することができます ([Configuration - Ansible Lint Documentation](https://ansible.readthedocs.io/projects/lint/configuring/#ansible-lint-configuration)).

今回紹介している [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) は package 化しているため `pip install` でのみで利用することができます.

例として既出の `playbooks/roles/sample/tasks/sample.yml`に対して ansible-lint を実行してみます.

```yaml
---
- name: Overwrite parent playbook vars
  ansible.builtin.set_fact:
    var__overwrite: overwritten (by <some_role>/tasks/sample.yml)
```

するとと以下のように検知されます.

```log
ansible-lint-custom-strict-naming<var_name_prefix>: Variables in 'set_fact' should have a 'sample_role__var__' prefix.
playbooks/roles/sample/tasks/sample.yml:2 Task/Handler: Overwrite parent playbook vars
```

[Ansible - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=redhat.ansible) を利用して VSCode で表示すると以下のようになります.

![Image from Gyazo](https://i.gyazo.com/7f0110078cf740753d3b5840a8017a32.png)

これで自分もチームも迷うこと無く変数を扱うことができますよね (圧).

## まとめ

- Ansible は便利で用途も広い
- しかし, 変数のスコープが無いため, 予期せず変数を上書きしてしまう等の問題点もある
- 一部の問題点は命名規則で改善することはできる
- ansible-lint のカスタムルールを作ることができれば変数の命名規則を自分やチームに強制することができる

## 最後に

- 明日 4 日目の記事は trdr さんの記事です. お楽しみに!!
- 本記事のサンプル Ansible プロジェクトは以下にあります
  - <https://github.com/pollenjp/2023-12-03-article-sample>
- [ansible-lint-custom-strict-naming · PyPI](https://pypi.org/project/ansible-lint-custom-strict-naming/) について
  - 本記事で紹介したルールを実装したパッケージです
  - パッケージの開発は雰囲気でやっており, 試行錯誤段階にあるため, そのまま利用される場合は package バージョンを固定することをお勧めします.
- なお, Ansible は様々な使われ方ができるため, お使いのプロジェクトに合わせて適切なルールを利用することをお勧めします.
