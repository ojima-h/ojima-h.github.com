---
layout: post
title: "emacs-overlay で Elisp をパッケージ管理"
date: 2012-05-02 22:59
comments: true
categories: [gentoo, portage, emacs, elisp]
---

関西 Emacs で Elisp のパッケージ管理の話が出てきたことだし、
portage で Elisp を管理してみることにする。

こういうのって、まずは Overlay から見てみるものなのかな...

ということで、emacs-overlay を使ってみる。

参考: <http://overlays.gentoo.org/proj/emacs/>

1.  layman をインストール

        emerge layman
		
2.  emacs-overlay を追加

        layman -a emacs
		
3.  layman をソースに追加。

        echo "source /var/lib/layman/make.conf" >> /etc/make.conf
		


`/var/lib/layman/emacs/app-emacs/` とかを見るとインストールできる Elisp が見える。

anything とか evil とかあるけど、、、、  
あんまり数は無いのね....

-----

ちなみに大学のプロキシ内からだと、svn プロトコルが通らずハマったけれど、
それはまた別のお話。

[emacs-overlay で Elisp をパッケージ管理 その2](/blog/2012/05/03/gentoo-emacs-overlay-2/)
