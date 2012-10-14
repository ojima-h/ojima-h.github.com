---
layout: page
title: "準備"
date: 2012-10-14 20:23
comments: true
sharing: true
footer: true
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

    `$ ctags -e` でemacs用のタグファイルを生成し、
    `M-.` でタグジャンプできます。
    
-   rdefs (gem)

-   rinari (elisp)

    packege-install ではいります。
    rails 用の統合開発環境みたいなかんじです。
    
-   ruby-debug

    <http://jampin.blog20.fc2.com/blog-entry-18.html> を参考に。
