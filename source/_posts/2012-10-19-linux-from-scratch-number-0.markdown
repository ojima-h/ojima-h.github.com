---
layout: post
title: "Linux from Scratch 始めます"
date: 2012-10-19 18:08
comments: true
categories: linux-from-scratch inov2013
---

前々からやる、やる、と言っていた Linux from Scratch に手をつけようと思います。

Linux from scratch (LFS) ていうのは、

>   Linux From Scratch(リナックス フロム スクラッチ、LFS)とは、Linuxを1から構築しようという試みである。すなわち、カーネルなども含めてすべて手作業でソースコードからコンパイルして作り上げる。
>    
>   具体的には、まず現在動作しているLinuxシステム上で、LFSをコンパイルするためのシステムを構築する。そのシステム上で、実際に稼働するLFSのシステムをコンパイルし、実際に起動することが出来る状態にまでもって行くという手順を踏む。
>   
>   ref. http://ja.wikipedia.org/wiki/Linux_from_Scratch

ということです。でもって、

>   LFS teaches people how a Linux system works internally
>   Building LFS teaches you about all that makes Linux tick, how things work together and depend on each other. And most importantly, how to customize it to your own tastes and needs. 
>   
>   ref. http://www.linuxfromscratch.org/lfs/

楽しく勉強できる、ということのようです。

本家 : <http://www.linuxfromscratch.org>

準備
----------

<http://www.linuxfromscratch.org/lfs/downloads/stable/> から、LFS-BOOK-7.2.pdf もしくは、LFS-BOOK-7.2.tar.bz (HTMLが入ってる)をダウンロードします。このBOOKに従って粛々とビルドしていきます。

次に、適当なマシンにLinuxをインストールします。LFSやるなら、32bitマシンが無難ぽいです。

というわけで、Core 2 Quad のマシンにGentooをインストールしました。安心の最小構成です。
手元のノートからSSHでアクセスすることにします。

Gentooがインストールされたシステムを「ホスト」と呼びます。
ホスト側で最小限のシステムをビルドした後、新しく構築した側のシステムに移りビスドを進めていくという
流れです。


パーティション作成
----------

LFSをビルドするために、新しいのパーティションを作ります。今回は10GBとしました。
スワップ領域、ブートパーティションはホストと共有することにました。

{% highlight console %}
$ fdisk /dev/sda4
{% endhighlight %}

新しくつくったパーティションを`/mnt/lfs`以下にマウントし、そこにソースをダウンロードしてきます。

ダウンロードするべきソースのリストは、<http://www.linuxfromscratch.org/lfs/downloads/stable/LFS-BOOK-7.2.tar.bz2> に含まれる wget-list にあります。md5sum も含まれています。

{% highlight console %}
$ wget -i wget-list -P /mnt/lfs/source
$ cd /mnt/lfs/source
$ md5sum -c ~/md5sums
{% endhighlight %}

あとはユーザを作って、環境変数を調整したり... 詳しくはLFS-BOOK参照。


これで、ソースが手に入ったので、あとはひたすらビルドしてくだけなんですかね...？そうだといいですね。

とりあえず、今日はここまで。


