# Ansibleとは

先程, Vagrantの練習問題で簡単なWebアプリケーションの環境構築をVagrant上に｢手動で｣行いました.
練習問題では, 構築する台数が1台だったので何とかなりましたが, もし同じ環境を10台作って欲しい, と言われたらどう思いますか?
手動で行えば, 10倍の時間がかかりますし, その間にコマンドの入力ミス(オペレーションミス, オペミス)など起こしてしまう可能性も十分あるでしょう.

こんな時に役立つのが, 環境構築を自動的に行ってくれる｢プロビジョニングツール｣と呼ばれるツール類です(｢構成管理ツール｣と呼ぶこともあります).
プロビジョニングツールを使えば, 構築手順(構築手順を記述する方法は, プロビジョニングツールによって異なっています)をコードとして書いてしまいさえすえば, それを複数のサーバに適用することが出来るようになります.
エンジニアは, プロビジョニングツールに対して環境構築を行うように命令するだけで済むので, 手動で構築する際に起こりえるオペレーションミスも防ぎやすいです.

近年, サービスのインフラストラクチャをコードで表現/実現する｢Infrastructure as a Code｣が流行していますが, プロビジョニングツールはその中の｢環境構築｣という位置領域を担うツールと言うことができます.

後述しますが, ｢Infrastructure as a Code｣の流行によって, 近年様々なプロビジョニングツールが産まれています.
そのため, 各社/各チームにとって適切なプロビジョニングツールを選択していく必要性があるでしょう.
とりあえず今回は, 両者で利用実績のあるAnsibleを使って, プロビジョニングツールとInfrastructure as a Codeを体験していきます.
具体的には, Ansibleの基本的な機能を解説しつつ, 先程の練習問題で手動で行った環境構築を自動的に行う構築手順を作成していきます.

## 代表的なプロビジョニングツール

|名前     |特徴                                                                                                               |
|:--------|:------------------------------------------------------------------------------------------------------------------|
| Ansible | 今回紹介するプロビジョニングツール. Python製. 構築手順はYAMLで記述する. 他のツールと比べると利用までの障壁は低い. |
| Chef    | Ruby製. 構築手順は｢レシピ｣と呼ばれ, Rubyで記述する. プロビジョニングツールとしては多機能.                         |
| Fabric  | Python製. 構築手順はPythonで記述する. Ansibleよりもプログラマブルに構築手順を記述できる. Chefよりはシンプル.      |
| Itamae  | Ruby製. Rubyで構築手順を記述でき, Chefよりもシンプルなプロビジョニングツール.                                     |

この他, プロビジョニングツールで構築した環境に, アプリケーションをデプロイするためのデプロイツールと呼ばれるツールもあります.
例えば, Capistrano(Ruby)やCinnamon(Perl)などが有名ですが, これらのデプロイツールとプロビジョニングツールの境界はかなり曖昧です.
実際にGaiaXでは, プロビジョニングツールのAnsibleで, プロビジョニングだけでなくデプロイも行う, という事例があります.

# Ansibleのインストール

Homebrewからインストールすることができます.

```
$ brew install ansible
```

Python製のツールなので, `easy_install`や`pip`でもインストールできます.
ただ, Mac OS XであればHomebrewでインストールするのが一番手っ取り早いですし, 安心でしょう.

# Ansible入門

## 仮想マシンの準備

それでは, まずAnsibleで架橋構築を行う対象となる仮想マシンを作っていきましょう.

```
$ vagrant init ubuntu/trusty64
$ vagrant up
```

これまで学んできたように, これで仮想マシンの準備は完了です.

Vagrantを利用するのであれば, `ssh`コマンド経由でVagrantに接続出来るようにしておくと楽になります.
`ssh-config`で, この仮想マシンに接続するために必要な設定を出力し, `.ssh/config`に追記しましょう.

```
$ vagrant ssh-config --host vagrant >> ~/.ssh/config
```

接続出来るか確認してみましょう.

```
$ ssh vagrant
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0-48-generic x86_64)
    ... 中略 ...
vagrant@vagrant-ubuntu-trusty-64:~$
```

問題ないですね.

## `hosts`とPlaybookの準備

Ansibleで環境構築を行うためには, 予めいくつかの準備が必要となります.

まずは`hosts`ファイルです.
このファイルには, 環境構築を行う対象となるホストを, IPないしホスト名で記述していきます.
ファイルはini形式で記載することができ, ホストのロール(役割)に応じてホストの集合をグループとして定義することもできます.

例えば, アプリケーションを動かすサーバが3台(`app1`〜`app3`), データベース用のサーバが2台(`db1`〜`db2`)があるのであれば, 次のように書くと良いでしょう.

```
[app]
app1
app2
app3

[db]
db1
db2
```

今回は, Amon2::Liteで作成した簡単なアプリケーションを動かすサーバが1台だけなので, 次のように設定しておきます.

```
[app]
vagrant
```

### Playbook

Playbookは, Ansibleにおける｢構築手順書｣そのものです.
先程設定した`hosts`に記載したホストやグループに, Playbookを適用することで, Ansibleは環境構築を行います.

Ansibleでは, PlaybookはYAML形式で記述します.
まずは試しに, 仮想マシンの`/tmp`ディレクトリに, ｢Hello, Ansible!｣と書かれた｢ansible.txt｣というファイルを設置するようなPlaybookを書いてみることとします.

Playbookの書き方についての詳細は後述します.
まずはとりあえず, このPlaybookをVagrantで作った仮想マシンに適用するところまでをやってみましょう.

```
- hosts: app
  sudo: yes
  tasks:
    - name: Hello, Ansible!
      shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
```

これを, `sample-playbook.yml`というファイル名で保存しておきます.
`hosts`ファイルとPlaybookファイルの設置が終わっていれば, カレントディレクトリは次のようになっているでしょう.

```
$ tree
.
├── Vagrantfile
├── hosts
└── sample-playbook.yml

0 directories, 3 files
```

## Playbookの実行

設定した`hosts`ファイルを使って, Playbook(`sample-playbook.yml`)を仮想マシンに適用してみましょう.

```
$ ansible-playbook -i hosts sample-playbook.yml
```

Playbookの適用は, `ansible-playbook`コマンドを利用します.
`-i`で`hosts`ファイルを指定し, その後にPlaybookのファイル名を指定します.

```
$ ansible-playbook -i hosts sample-playbook.yml

PLAY [app] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [vagrant]

TASK: [Hello, Ansible!] *******************************************************
changed: [vagrant]

PLAY RECAP ********************************************************************
vagrant                    : ok=2    changed=1    unreachable=0    failed=0
```

最後に, 実行結果が出力されています.

```
PLAY RECAP ********************************************************************
vagrant                    : ok=2    changed=1    unreachable=0    failed=0
```

タスクを処理するごとに, Playbookの処理が成功したら`ok`, 変更を伴う処理が成功すれば`changed`, ホストに接続できなければ`unreachable`, そして失敗した場合は`failed`が, それぞれ1ずつ増えていきます.
そのため, `unreachable`と`failed`が0であれば, Playbookの適用は成功した, ということになります.

それでは, 本当に仮想マシン上にファイルが生成されたのか, 仮想マシンに接続して確認してみましょう.

```
$ ssh vagrant
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0-48-generic x86_64)
    ... 中略 ...
vagrant@vagrant-ubuntu-trusty-64:~$ cat /tmp/ansible.txt
Hello, Ansible!
```

...確かに, 生成されていますね!

ここまで, 簡単なPlaybookを通して, Ansibleを使った環境構築の自動化を体験してきました.
今回は, ただファイルを設置する簡単なPlaybookだったので, そこまで大きいメリットを感じなかったかもしれません.
ですが, 実際にアプリケーションを動かすための環境を構築するためには, これまで体験してきたように様々なツールやミドルウェアをホストに導入し, その設定ファイルをホスト内に適切に設置していかなければなりません.

Ansibleのようなプロビジョニングツールが真価を発揮するのは, まさにこのような場面です.
そしてこのような場面は, 私達がエンジニアとして仕事をしている中では, もはや日常茶飯事と言うことが出来るでしょう.
そういった意味で, Ansibleのようなプロビジョニングツールやデプロイツールの知識は, 恐らくこれからのエンジニアにとって必須の知識になってくると思います.
なので, まずはこのカリキュラムで, しっかりAnsibleに慣れてしまいましょう.

...それでは次に, Ansibleの｢キモ｣となる, Playbookの書き方について, さらに細かく見ていきます.

## Playbookの書き方

先程述べたように, AnsibleのPlaybookは次のようなYAML形式で記述します.

```
- hosts: app
  sudo: yes
  tasks:
    - name: Hello, Ansible!
      shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
```

Playbookで設定出来る項目のうち, いくつかを紹介します.

### hosts

`ansible-playbook`コマンドに対し`-i`オプションで指定したホストファイルのうち, このPlaybookを適用する(実行する)ホストやグループを指定します.
カンマ区切りやYAMLのリスト指定で, 同時に複数のホストやグループを指定できます.

### sudo

`sudo`が`yes`の場合, このPlaybookはホスト側では`sudo`を使って実行されます.
デフォルトでは`root`として実行しますが, `sudo_user`を指定すれば任意のユーザとして実行することもできます.

### gather_facts

`yes`の場合, あるいは無指定の場合, Playbookを実行するを実行する前に対象となるホストの情報を取得します(取得できる情報については, ｢Ansible Tutorial｣の[対象ホストの情報を取得(GATHERING FACTS)](http://yteraoka.github.io/ansible-tutorial/#gather-facts)などを参考にしてください).
取得した情報は, Playbook中で参照することも出来るので, 例えば｢OSや, そのバージョンに応じて, 実行する処理を変更する｣などといった事も可能です.

ホスト情報の取得には少し時間がかかりますので, ホスト情報を利用しない場合は`gather_facts`を`no`にすることで, Playbookを実行する時間を少し減らすことが出来ます.

### tasks

ホストで実行したい処理を記述することができます.

```
  - tasks:
    - name: Hello, Ansible!
      shell: echo 'Hello, Ansible!' > /tmp/ansible1.txt
```

タスクは, Ansibleが提供する｢モジュール｣を使って記述します(英語ですが, モジュールの一覧は[こちら](http://docs.ansible.com/list_of_all_modules.html)にあります).
ここでは, ホスト側で任意のコマンドを実行する[shell](http://docs.ansible.com/shell_module.html)というモジュールを実行しています.

タスクを指定する際は, 別途`name`でその解説を記入することが出来ます.
`name`を指定した場合, `ansible-playbook`でPlaybookを実行した際に, 次のように表示されます.

```
TASK: [Hello, Ansible!] *******************************************************
```

実行結果の可読性が向上するので, 是非指定するようにしておきましょう.

なお, Playbookで利用できるモジュールのうち, 代表的なものについては, この後でいくつか紹介します.

### vars_files

Playbook中で利用できる変数を指定できます.
例えば, `vars.yml`というファイルに,

```
word: 'Hello, Ansible!'
```

のように指定しておけば, Playbookの中から`Hello, Ansible!`という文字列を, `{{ word }}`として参照することができます.

```
- hosts: app
  sudo: yes
  vars_files:
    - vars.yml
  tasks:
    - name: Hello, Ansible!
      shell: echo '{{ word }}' > /tmp/ansible.txt
```

例えばこのようなPlaybookを実行した場合, ホスト側の`/tmp/ansible.txt`には, ｢{{ word }}｣ではなく｢word｣を`vars.yml`で展開した｢Hello, Ansible!｣が書き込まれます.
変数は, ここで紹介している`vars_files`で指定したものの他に, デフォルトで利用できる[マジック変数](http://qiita.com/h2suzuki/items/15609e0de4a2402803e9)というものもあります.

## 代表的なモジュール

### [yum](http://docs.ansible.com/yum_module.html) / [apt](http://docs.ansible.com/apt_module.html)

```
- apt: name=foo
- yum: name=foo
```

`yum`コマンドや`apt-get`コマンドを使って, `name`で指定したパッケージをインストールします.

### [shell](http://docs.ansible.com/shell_module.html)

```
- shell: echo 'sample' >> log.txt
```

指定したコマンドを, ホスト側で実行します.

### [file](http://docs.ansible.com/file_module.html)

```
- file: path=/tmp/sample state=directory
- file: path=/tmp/sample/log state=touch
```

ファイルやディレクトリの操作をします. `state`が｢directory｣の場合, `path`で指定したディレクトリを作成し, `state`が｢touch｣の場合は空のファイルを作成することができます.

### [service](http://docs.ansible.com/service_module.html)

```
- service: name=nginx state=started
- service: name=nginx state=started enabled=yes
```

`name`で指定したパッケージを操作することができます.
`state`で, パッケージの動作状況を変更することができ, `started`で起動, `stopped`で停止, `restarted`で再起動することができます.

また, `enabled`でブート時に自動起動するかを設定することができます(`enabled=yes`でブート時に自動的に起動するようになる).

### [git](http://docs.ansible.com/git_module.html)

```
- git: repo=
```

### [copy](http://docs.ansible.com/copy_module.html)

```
- copy: src=file.txt dest=/tmp/file.txt
```

ローカルマシンにある, `src`で指定したファイルを, ホストの`dest`で指定したパスにコピーすることができます.
ライブラリやアプリケーションの設定ファイルの設置などに利用することができます.

### [template](http://docs.ansible.com/template_module.html)

```
- template: src=file.txt dest=/tmp/file.txt
```

ローカルマシンにある, `src`で指定したテンプレートを元にファイルを生成し, ホストの`dest`で指定したパスにコピーすることができます.
`copy`モジュールとの違いは, `src`で指定したファイルをテンプレートとして, 動的にファイルを生成できる点です.

例えば, `vars_files`で`type`という変数で`Ansible`を指定している場合, `file.txt`の中身が

```
Hello, {{ type }}!
```

このようになっているのであれば, `dest`で指定した`/tmp/file.txt`の中身は, 次のようになります.

```
Hello, Ansible!
```

templateモジュールはPythonのJinja2というテンプレートエンジン(日本語マニュアルは[こちら](http://ymotongpoo.appspot.com/jinja2_ja/index.html))を利用しているので, Ninja2の構文に従って条件分岐や繰り返しなども実現することができます.

## コラム: Playbookのシンタックスチェック

AnsibleのPlaybookのシンタックスチェックは, `ansible-playbook`に対して`--syntax-check`オプションを指定すればOKです.
シンタックスエラーがなければ, 次のような出力になります.

```
$ ansible-playbook -i hosts sample-playbook.yml --syntax-check

playbook: sample-playbook.yml
```

一方, シンタックスエラーが発生した場合はその詳細が出力されます.
例えば, Playbookの内容が次のようになっている場合...

```
- hosts: app
  sudo: yes
  tasks:
    - name: Hello, Ansible!
    shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
```

シンタックスチェックの結果は, 次の通りになります.

```
$ ansible-playbook -i hosts sample-playbook.yml --syntax-check
ERROR: Syntax Error while loading YAML script, sample-playbook.yml
Note: The error may actually appear before this position: line 5, column 5

    - name: Hello, Ansible!
    shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
    ^
```

`name`と`shell`の行頭を揃えていないのでシンタックスエラーが生じていることがわかります.

## 練習問題

Vagrantの｢練習問題｣において手動で行った環境構築を, Ansibleを利用して自動化してみましょう(Amon2::Liteアプリケーションの準備や, Vagrantfileの用意と起動などは手動で行ってください).

```
- hosts: app
  sudo: yes
  gather_facts: no
  tasks:
    - name: Install git
      apt: name=git
    - name: Clone xbuild
      git: repo=https://github.com/tagomoris/xbuild.git dest=/tmp/xbuild
    - name: Install Perl (and App::cpanminus and Carton)
      shell: /tmp/xbuild/perl-install 5.20.1 /opt/perl-5.20
```

ヒントとして, xbuildを利用してPerl 5.20.1をインストールするまでのPlaybookを用意しておきました.
このPlaybookをベースに, `carton install`の実行やNginx, Supervisorのインストールと初期設定などをAnsibleで自動的に実行できるようにしてみましょう.

## 複雑なPlaybook

...

## 練習問題

...

## まとめ

ここで紹介したように, Ansibleを利用して環境構築の手順をPlaybookに落とし込んでおけば, 手動より遥かに高速に構築することが出来ますし, 何より｢ノウハウを持っている人でないと, 環境構築ができない｣という"属人性"を排除することができます.

個人的な意見ではありますが, チームで開発を進める上で, 特に環境構築やデプロイの属人化は出来る限り避けるべきだと思います.
もし環境構築やデプロイが属人化してしまえば, そのスキルとノウハウを持っている人の負担が増えてしまいますし, その人がいなければチームの仕事が回らない, という状況に陥ってしまいます.

近年, ｢アジャイル開発｣などが注目され, 開発とリリースの素早いサイクルが求められつつあります.
そういう場面において, チームメンバーの誰もが環境構築やデプロイを出来るような環境というのは, 必要不可欠と言うことが出来ます.
そのため, これからはAnsibleや後述するServerspecのような, サービスのインフラをコードで表現, 記述し, 可読化を実現する｢Infrastructure as a Code｣を実現する為のツールが重要になってくるでしょう.

次は, Ansibleなどのプロビジョニングツールを使って構築した環境が, 本当に正しいのかを検証する為のツール, Serverspecについて学んでいきます.
これで, プロビジョニングツールによる｢環境構築｣が終わった後に, その環境を｢検証(テスト)｣することが出来るようになります.