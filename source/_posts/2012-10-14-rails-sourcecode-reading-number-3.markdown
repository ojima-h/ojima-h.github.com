---
layout: post
title: "rails sourcecode reading #3 -- 準備"
date: 2012-10-14 23:20
comments: true
categories: rails-sourcecode-reading inov2013
---

まずは、ソースコードを用意しました。
bundle を利用することにしました。

    $ mkdir rails_sourcecode_reading
    $ cd rails_sourcecode_reading
    $ mkdir src
    
    $ vi Gemfile
    
    -----------------------------
    # Gemfile
    source "http://rubygems.org"
     
    gem "rails", "3.2.8"
    -----------------------------
    
    $ bundle install --path="./src"
    
これで、src/ruby/1.9.1/gems 以下に、railsに関連するgemたちが全部入ります。



次に、ソースコードを読む環境を整えました。
もちろん、emacsで読んでいきます。
以下は、利用した elisp とか gem とかです。

-   ctags

    `ctags -e -R . --langmap=Ruby:.rb --ruby-types=cfFm` でemacs用のタグファイルを生成し、
    `M-.` でタグジャンプできます。

    参考 : <http://wiki.livedoor.jp/koziy/d/ruby/ctags>
    
-   rdefs (gem)

-   rinari (elisp)

    packege-install ではいります。
    rails 用の統合開発環境みたいなかんじです。
    
-   ruby-debug

    <http://jampin.blog20.fc2.com/blog-entry-18.html> を参考に。

-   yard

    `yard server` でWebサーバ起動


まずは、ActiveRecord と Arel あたりから読んでいこうと思います。
