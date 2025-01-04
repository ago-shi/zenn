---
title: "ansibleを使ってraid1を構成した所感"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["raid1", "linux", "ansible"]
published: true
---
## この記事の目的
- 構成管理にansibleを使う場合の難しさを共有
- linuxでraid1を構成した際の例を基に、工夫点を共有

## ansibleでハードウェアを扱うのは難しい
サーバ構築にansibleを利用する際は、冪等性を意識してplaybookを書いている。

`<<基本方針>>`
`初期構築時も、構築後も、何度ansibleを実行しても同じ状態になる。`

大体は基本方針に従って書けるが、raidを構成しようとしたときに難関にぶつかった。

``ディスクデバイス名はOS起動毎に変わる可能性がある。``

初期構築時に/dev/sdaと/dev/sdbでraidを構成したとして、構築後にOS再起動すると/dev/sdcと/dev/sddがraidを構成したデバイスになっている可能性がある。

playbook上に/dev/sda, /dev/sdbを定義してしまうと、初期構築時は良いが、構築後に実行するとエラーになる可能性がある。

`<<エラーの例>>`
`/dev/sdb: no such file or directory`

エラー終了なら良いが、別の用途で使用しているディスクでraidを構成してしまい、元のデータを消去してしまうリスクがある。(playbookの書き方による。)

ansibleはOSをセットアップする構成管理ツールと考えた方が良い。
ハードウェアはOSの外の世界であり、そんなハードウェアを意識する様な制御には向いていないということだと思う。

## それでもansibleでraidを構成したい(playbookの工夫)
サーバ構築時にどうしてもハードウェアを意識するケースで、極力基本方針に従ってplaybookを書くにはどうしたら良いか？

初期構築時と構築後でtaskを分岐させることを考えた。
- 初期構築時はディスクパーティション作成とraidを構成するtaskを通す。
- 構築後は該当のtaskを通さない。

```yaml: playbook抜粋
# ディスクパーティション作成taskを呼び出す
- name: include disk partition create tasks
  ansible.builtin.include_tasks: diskPartitionCreate.yml
  ## 初期構築時はpartBuildReq, raidConf4Buildを定義しておく
  when: >-
    (partBuildReq | default(false)) and 
    (raidConf4Build is defined | default(false))

# raid構成taskを呼び出す
- name: include raid1 build tasks
  ansible.builtin.include_tasks: raid1Build.yml
  ## 初期構築時はraid1BuildReq, raidConf4Buildを定義しておく
  when: >-
    (raid1BuildReq | default(false)) and 
    (raidConf4Build is defined | default(false))
```

"partBuildReq", "raid1BuildReq", "raidConf4Build"というパラメータはansible-playbookコマンド実行時に-eオプションで定義する/しないで初期構築と構築後のansible実行をコントロールする。

```yaml:ansible-playbookコマンド
# 初期構築時
ansible-playbook -i inventory.ini playbook.yml -e @exRaidDevice.yml

# 構築後
ansible-playbook -i inventory.ini playbook.yml
```

構築後のansible実行ではハードウェアの操作をスキップしているため、常に同じplaybookを実行すれば良いものの、基本方針に則っていると言い切るのは正直憚られる。

とは言え、前述の通りansibleはOSセットアップ用の構成管理ツールと考えているのでここら辺が落としどころで良いと思っている。(妥協したとも言える。)

## 補足
1. 例示したplaybookは以下のnfs.yml
nfsサーバのディスクとしてraid1を構成したので、このplaybookを実行するとnfsサーバが出来る。
https://github.com/ago-shi/uws
