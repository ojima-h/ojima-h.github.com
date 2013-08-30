---
layout: post
title: "Hadoop のチューニングに関してまとめ"
date: 2013-08-29 16:06
comments: true
categories: [Hadoop, Hive]
---

1効率の良い HiveQL を書くときに気をつけるべきことをまとめてみる。

_以下に書いたことは、ほとんど全部自分なりの解釈なので、間違いが多々あると思います_

Hadoop一般のチューニングに関しては、ここを参考にした。

<http://www.cloudera.co.jp/jpevents/cloudera-world-tokyo/pdf/A3_Hadoopのシステム設計_運用のポイント.pdf>

[Hadoop本](http://www.amazon.co.jp/Hadoop-Tom-White/dp/487311439X) を参考に、この資料の補足してみた↓

ラック内ネットワークとラック間ネットワークは何が違うのか？
-----------------------------

_(p.69 「ネットワークトポロジとHadoop」 / p.267 9.1.1 ネットワークトポロジ)_

Hadoop は、データを読み書きする際にネットワークのトポロジやネットワーク構成を勘案しながら動作する。
読み込みを行う際には、クライアントになるべく _近い_ ノードから読み込むように調整するし、
書き込みを行う際には、データのレプリカが近い場所に集中してしまわないように調整する。

2つのノード間のデータ転送速度が早いほど、この2つのノードが _近い_ と見做される。
Hadoop は、ノード間の距離をネットワーク構成から算出している。
つまり、「同じラックにある2つのノードは互いに近い = 転送速度が早い」ものとして処理を進める。

そのため、同じラック内にあるのに実際には転送速度が遅い、ということになると、
Hadoop が効率良くデータを処理することができなくなってしまう。

スレーブノードでは、なぜ RAID0 を組んではダメなのか？
------------------------------

_(p.266 「RAIDを使わないのはなぜ？」)_

- HDFSはノード間の複製によって冗長性を持つので、RAIDで冗長性を提供してもらう必要がない。(←弱い)
- RAID0 でデータロストが発生した事例あり

マスターノードでRaidを組まないと何が起きるのか？
--------------------------

マスターノード (= ネームノード) は、クラスタ上のどのノードにどのファイルのデータが保存されているかを管理している。
マスターノードのデータが壊れると、HDFSからファイルデータを読み取ることができなくなってしまう。

そのため、マスターノードは信頼性を確保しておく必要がある。

マスターとスレーブで、求められるCPU性能が違うのはなぜ？
-----------------------------

実際に計算するのはスレーブノード。
マスターノードは、クライアントの要求を受けて適切なスレーブノードに指令を出すのが仕事。
だから、マスターノードはたいしてCPU性能を必要としない。

ECCとはなんぞ？
---------

[Error Check and Correct memory](http://e-words.jp/w/ECCE383A1E383A2E383AA.html)

> メモリに誤った値が記録されていることを検出し、正しい値に訂正することができるメモリモジュール。

なんで圧縮したら「速度」があがるの？
------------------

マップタスクからの出力は一時的にディスクに保存される。
レデュースタスクからの出力もディスクに保存される。

一般的には、Hadoopはかなり大きなデータを扱うので、出力結果を圧縮しないままディスクに書き込む時間よりも、
(圧縮する時間 + 圧縮データをディスクに書き込む時間) の方が短くなる。
読み込む時も同様に、非圧縮のデータを読み出す時間よりも、(圧縮されたデータを読み出す時間 + 解凍する時間) の方が短かくなる。

ブロック数(サイズ)とヒープサイズの関係は？ (16枚目)
-----------------------------

ブロック数が小さい = ブロックサイズが大きい = ディスクから1回に読み込むデータのサイズが大きい = ディスクへのアクセスが減る

ヒープサイズが大きい = 計算結果を沢山メモリに置いておける = 計算結果をディスクに書き出すことが減る = ディスクへのアクセスが減る

⇒ **速い**

フェデレーション？て何？
------------

<http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/Federation.html>

> The prior HDFS architecture allows only a single namespace for the entire cluster. A single Namenode manages this namespace. HDFS Federation addresses limitation of the prior architecture by adding support multiple Namenodes/namespaces to HDFS file system.

> In order to scale the name service horizontally, federation uses multiple independent Namenodes/namespaces. The Namenodes are federated, that is, the Namenodes are independent and don’t require coordination with each other. The datanodes are used as common storage for blocks by all the Namenodes. Each datanode registers with all the Namenodes in the cluster. Datanodes send periodic heartbeats and block reports and handles commands from the Namenodes.

ネームノードの性能をスケールさせるために、HDFSファイル空間を2つに分けてしまって2つのネームノードでそれぞれ担当しよう、という話のようだ。

ネームノードの冗長化とはまた別の話なのかな？

「ディスクのマウントポイントごとに別々のディレクトリをカンマ区切りで指定すること」てどういう意味？(21枚目)
-------------------------------------------------------

1つのデータノードのストレージはいくつかのディスクで構成されている。
ディスクが複数あってそれぞれにデータが分散して保存されているから、
データノードは処理を並列して行うことができる。

で、データノードは `dfs.datanode.data.dir` に指定されたディレクトリに対してラウンドロビンで書き込んでいくから、
`dfs.datanode.data.dir` には、複数あるディスクのそれぞれのマウントポイントを指定しないとダメだよ、ということ。

ちなみに、JBOD っていうのは、<http://ja.wikipedia.org/wiki/JBOD> 参照。

「タクク数 > ディスク数」てどういう状態？(25枚目)
----------------------------

それぞれのタスクは、(普通は)ディスクからデータを読みとってから計算をはじめる。
「タクク数 > ディスク数」になってしまうと、並行してディスクアクセスを行なえるスレッド数(プロセス数？)の限界を越えて、
タスクノードがディスクアクセスを要求してくることになる。

で、ディスクアクセスが詰まってしまう。
