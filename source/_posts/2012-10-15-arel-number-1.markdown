---
layout: post
title: "arel #1"
date: 2012-10-15 00:00
comments: true
categories: 
---

ref. <http://gihyo.jp/dev/serial/01/ruby/0043>

arel-3.0.2、activerecord-3.2.8 を使ってます。

目標は、次のコードの実行過程を理解すること。

    books = Arel::Table.new :books
    books = books.where(books[:title].eq('Head First Rails'))
    puts books.to_sql

* * * * *

Arel を使うためには、データベースとのコネクションを作る必要があるのですが、
その部分は ActiveRecord のおまかせします。

    require 'arel'
    require 'sqlite3'
    require 'active_record'
     
    ActiveRecord::Base.configurations = {'development' => {:adapter => 'sqlite3', :database => 'books.sqlite3'}}
    ActiveRecord::Base.establish_connection('development')
     
    Arel::Table.engine = Arel::Sql::Engine.new(ActiveRecord::Base)

* * * * *

さて、つらつらと読んでいくことにします。

でも、眠いので今日はここまで。


