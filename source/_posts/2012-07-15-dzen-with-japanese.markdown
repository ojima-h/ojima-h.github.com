---
layout: post
title: "dzenで日本語"
date: 2012-07-15 04:38
comments: true
categories: xmonad
---

dzen で日本語を表示するまでのまとめ。

portage で提供されている dzen をインストールしたところ、日本語が化けた。

どうやら、xft のサポートをつける必要があるみたい。

ところが、通常のレポジトリで提供されているソースは xft に対応していないようだ。

というわけで、<https://github.com/robm/dzen> から最新版をもってきて、普通に make してインストールしたら、表示できました。

ちなみに、フォントを指定してやらないと、かなり汚い。アンチエイリアスも設定できた。

    dzen2 -fn 'xft\
              \:IPAGothic\
              \:pixelsize=13\
              \:weight=regular\
              \:width=semicondensed\
              \:dpi=96\
              \:hinting=true\
              \:hintstyle=hintslight\
              \:antialias=true\
              \:rgba=rgb\
              \:lcdfilter=lcdlight\
              \'

一応ebuildをのっけてみる。。。

<https://github.com/ojima-h/my-overlays/blob/master/x11-misc/dzen/dzen-9999.ebuild>
