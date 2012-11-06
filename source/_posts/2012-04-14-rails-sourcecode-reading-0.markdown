---
layout: post
title: 第0回 Railsソースコードリーディング
date: 2012-04-14 23:31
comments: true
categories: rails-sourcecode-reading
---

なんとなくで始めみてみたRailsソースコードリーディング。
手探りながらの第0回でした。

まぁ、ぐだぐだ、でした


今日のお題は

**Railsの概要と基本的な使い方**


まずは、MVCに関するお話。  
MVCとは、アプリケーションの構成を

-   M (=Model)
    データを管理するコンポーネント
-   V(=View)
    ユーザの目の前に表示される部分を管理するコンポーネント
-   C(=controller)
    ModelとViewのあいだをとりもつコンポーネント

の3つに分けて捉えましょう、という考えかた。

一方で、Railsの構成がどうなっているかというと、、、

実はRails本体はからっぽで、実体は次のようなライブラリ群からできている。

-   ActionPack
    -   ActionController
	-   ActionView
	-   ActionDispatch
	
-   ActiveRecord

-   ActiveModel

-   ActiveResource

-   etc...

-   railties



そして、M,V,C のそれぞれと、 Railsの各構成要素のそれぞれが一対一に対応している。

-   Model <=> ActiveRecord, ActiveModel, ActiveResource
-   View <=> ActionView
-   Controller <=> ActionController, ActionDispatch

そしてそして、railtiesがこれらのライブラリ群をまとめあげて、コマンドラインインターフェイスを提供していると。。。


というわけで、次回以降はライブラリ群を1つ1つ読み解いていこう!! というお話でした。

とりあえず、次回はActiveRecordで。来週です。
	
	
--------

さてさて、Railsの簡単な使いかたを勉強しましょう、ということになりました。

    rails new sample
	
からはじめて、

    rails generate scaffold pages title:string content:text

ここまでは何の問題もなかったんです。

ところが

    rake db:migrate
	
したところ、、、

__全員エラー!!!__

しかも、エラーメッセージが意味不明だし。


そんなこんなしているうちに時間がきてしまい、結局グダグダになってしまいました。




はぁ、書くの飽きた。。。




まぁ、そんなかんじの第0回でした。
