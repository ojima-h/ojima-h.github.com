---
layout: post
title: "showoff 使ってみた"
date: 2012-04-22 14:00
comments: true
categories: [ruby,showoff,slide]
---

<https://github.com/schacon/showoff>

Ruby + Markdown で HTML なスライドをつくれる、ということで、使ってみた。

感想は

-   Markdown やっぱり素敵
-   Markdown から直にプレビューできる
-   コマンド一発で github page に公開できるのが便利
-   ある程度きれいにみせようと思ったら、CSS と javascript がんばって書
    かないといけない
-   一枚のスライドのなかで内容を少しづつ見せる、ってことができなかった

てかんじでした。

- - - - - - - - - 

使うまでの記録を書いておく。

とりあえず使う
----------

まずはインストール。

	gem install showoff
	
新規プレゼンテーションの作成

	showoff create new_presentation
	cd new_presentation
	
すると、"one" というディレクトリの下に "01_slide.md" というファイルが
できているはず。

この "01_slide.md" にマークダウンで色々書いていけばよい。

`!SLIDE` という行があると、そこでスライドが切り替わる。

コマンドラインから

	$ showoff serve
	
を実行し、`localhost:9090` にアクセスすると、プレビューを見ることがで
きる。

ちなみに、Firefox + Vimperator を使っているとスライドの操作ができない
ので注意。`Shift + escape` で Vimperator のキーを無効にすれば大丈夫。

さらに、

	$ showoff static
	
で、"static" というディレクトリができて、その中に整形された HTML が吐かれる。


詳しい使いかたは <https://github.com/schacon/showoff> を参照。


github pages への公開方法
----------

まずは、ソース全体を公開するためのレポジトリを作成。
全部のファイルをコミットする。

そしたら、おもむろに

	$ showoff github
	
これで、"gh_pages" というブランチに、`showoff static` されたコードをコ
ミットしてくれる。

あとは、`git push orgin gh-pages` するだけで、
"username.github.com/repository-name/" にスライドが公開される。

- - - - - - - - - -

うーん、やっぱりちょっとづつ表示していくのが欲しいな....
javascript でできるのかな...
