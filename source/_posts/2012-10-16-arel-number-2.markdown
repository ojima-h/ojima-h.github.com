---
layout: post
title: "arel #2"
date: 2012-10-16 03:30
comments: true
categories: 
---

arel-3.0.2 での以下のコードの実行過程を追い掛けています。

    1: books = Arel::Table.new :books
    2: books = books.where(books[:title].eq('Head First Rails'))
    3: puts books.to_sql


`Table#where`
----------

`books.where` を呼びだすと、新たに `SelectManager`クラスのインスタンスが生成され、
そのインスタンスが保持する`wheres`というリストに、引数が追加されます。

`Table#[]`
----------

次に、`books[:title]` の部分について見ます。

`Table::[]`は、`Arel::Attribute.new`を呼びだしています。


### `Attribute` と `Struct` について

`Struct`とは、、、

>   構造体クラス。Struct.new はこのクラスのサブクラスを新たに生成します。
>   個々の構造体はサブクラスから Struct.new を使って生成します。
>   
>   ref. <http://doc.ruby-lang.org/ja/1.9.3/class/Struct.html>

クラスを作るコンストラクタって何だよ...

arel/attributes/attribute.rb では以下のように使われています。

    module Arel
      module Attributes
        class Attribute < Struct.new :relation, :name
          include Arel::Expressions
          include Arel::Predications
          include Arel::AliasPredication
          include Arel::OrderPredications
          include Arel::Math
          .....
          
`Arel::Predications` では例えば次のようなメソッドを定義しています。

-   Arel::Predications#not_eq
-   Arel::Predications#not_eq_any
-   Arel::Predications#not_eq_all
-   Arel::Predications#eq
-   Arel::Predications#eq_any

つまり、`Attribute` という構造体(のサブクラス)に対して、比較や算術演算などができるようになっている、ということです。

`Table#[]` はこの拡張された構造体を返していることが分かります。


`Expressions#eq`
----------

`Expressions`は`Binary`を継承しています。

`Binary`のコンストラクタは、単に引数を記憶しておくだけです。

    class Binary
      def initialize left, right
        @left  = left
        @right = right
      end
    ...

`Table#where`と同様、引数を記憶しておくだけで、具体的な処理はまだ何もしていません。

### `Class.new` と `const_set`

話は逸れますが、binary.rbに以下のような記述がありました。

    module Nodes
      class Binary < Arel::Nodes::Node
      ...
      end

      %w{
        As
        Assignment
        Between
        ...
      }.each do |name|
        const_set name, Class.new(Binary)
      end
    end

`Module#const_set`は、モジュールに定数を定義するメソッドです。

`Class.new(supercass = Object)`は、`superclass`を親クラスとして匿名クラスをつくります。

Nodeモジュールに、Binaryクラスのサブクラスを値として、各種バイナリオペレータに対応する定数を定義しています。

ぱっとみエグいです。
