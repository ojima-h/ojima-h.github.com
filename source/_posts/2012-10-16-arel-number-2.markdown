---
layout: post
title: "arel #2"
date: 2012-10-16 03:30
comments: true
categories: rails-sourcecode-reading inov2013
---

arel-3.0.2 での以下のコードの実行過程を追い掛けています。

{% highlight ruby linenos%}
ActiveRecord::Base.configurations = {'development' => {:adapter => 'sqlite3', :database => 'books.sqlite3'}}
ActiveRecord::Base.establish_connection('development')
Arel::Table.engine = ActiveRecord::Base

books = Arel::Table.new :books
books = books.where(books[:title].eq('Head First Rails'))
puts books.to_sql
{% endhighlight %}

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

{% highlight ruby linenos %}
module Arel
  module Attributes
    class Attribute < Struct.new :relation, :name
      include Arel::Expressions
      include Arel::Predications
      include Arel::AliasPredication
      include Arel::OrderPredications
      include Arel::Math
      .....
    end

    Attribute = Attributes::Attribute
  end
end
{% endhighlight %}
          
`Arel::Predications` では例えば次のようなメソッドを定義しています。

-   Arel::Predications#not_eq
-   Arel::Predications#not_eq_any
-   Arel::Predications#not_eq_all
-   Arel::Predications#eq
-   Arel::Predications#eq_any

つまり、`Attribute` という構造体(のサブクラス)に対して、比較や算術演算などができるようになっている、ということです。

`Table#[]` はこの拡張された構造体を返していることが分かります。

{% highlight ruby linenos %}
  def [] name
    ::Arel::Attribute.new self, name
  end
{% endhighlight %}

つまり、`books[:title].relation`には`self`が、`books[:title].name`には`:title`が格納されます。
さらに、`books[:title]`は`Predications`モジュールから取り込まれた`eq`というメソッドをもっています。

ちなみに、`::Arel::Attribute.new`としていますが、`Arel::Attribute`は`Arel`モジュールの定数で、
実体は`Arel::Attributes::Attribute`クラスです。


`Predications#eq`
----------

`Predications#eq`は↓

{% highlight ruby linenos %}
  def eq other
    Nodes::Equality.new self, other
  end
{% endhighlight %}

`Nodes::Equality`クラスは、`Binary`を継承しています。

`Binary`のコンストラクタは、単に引数を記憶しておくだけです。

{% highlight ruby linenos %}
class Binary
  def initialize left, right
    @left  = left
    @right = right
  end
...
{% endhighlight %}

この時点では、具体的な処理はまだ何もしていません。

`eq`メソッドは`Arel::Nodes::Equalitiy`クラスのインスタンスを返します。

### `Class.new` と `const_set`

話は逸れますが、binary.rbに以下のような記述がありました。

{% highlight ruby linenos %}
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
{% endhighlight %}

`Module#const_set`は、モジュールに定数を定義するメソッドです。

`Class.new(supercass = Object)`は、`superclass`を親クラスとして匿名クラスをつくります。

Nodeモジュールに、Binaryクラスのサブクラスを値として、各種バイナリオペレータに対応する定数を定義しています。

ぱっとみエグいです。


`Table#where`
----------

`books.where` を呼びだすと、新たに `SelectManager`クラスのインスタンスが生成され、
そのインスタンスが保持する`wheres`というリストに、引数が追加されます。

`Expressions#eq`と同様、引数を記憶しておくだけです。


`SelectManager#to_sql`
----------

さて、具体的にSQLに変換している部分を見ていくことにします。

`Table#where` を実行すると、`SelectManager`クラスのインスタンスを返します。

`SelectManager#to_sql` の呼び出しは、継承元の `TreeManager`クラスに渡ります。


{% highlight ruby linenos %}
module Arel
  class TreeManager
    ...
    def visitor
      engine.connection.visitor
    end
 
    def to_sql
      visitor.accept @ast
    end
    ...
{% endhighlight %}

`engine.connection.visitor` ですが、この`engine`は最初に

{% highlight ruby linenos %}
Arel::Table.engine = ActiveRecord::Base
{% endhighlight %}

ででてきた`engine`です。

デバッガで、`visitor`が現れる場所を探していくと、

{% highlight ruby linenos %}
# /activerecord-3.2.8/lib/active_record/connection_adapters/sqlite_adapter.rb:83

  if config.fetch(:prepared_statements) { true }
    @visitor = Arel::Visitors::SQLite.new self
  else
{% endhighlight %}

に行きつきます。ActiveRecord側で何が起きているかは深く考えないことにして、
とにかく、`Arel::Visitors::SQLite` を見ていくことにします。

結局、`TreeManager#to_sql` は、`Arel::Visitors::ToSql#accept`に行き着くことになります。

ちなみに、`accept` の引数`@ast`は、`books.where`を呼びだした時に生成された `SelectManager`クラスのインスタンスで、これが`books.where`に渡された引数を保持しています。


`Visitors::Visitor`
----------

`Visitors::ToSql` は `Visitors::Visitor` を継承しています。

{% highlight ruby linenos %}
module Arel
  module Visitors
    class Visitor
      def accept object
        visit object
      end

      private

      DISPATCH = Hash.new do |hash, klass|
        hash[klass] = "visit_#{(klass.name || '').gsub('::', '_')}"
      end

      def dispatch
        DISPATCH
      end

      def visit object
        send dispatch[object.class], object
      rescue NoMethodError => e
        raise e if respond_to?(dispatch[object.class], true)
        superklass = object.class.ancestors.find { |klass|
          respond_to?(dispatch[klass], true)
        }
        raise(TypeError, "Cannot visit #{object.class}") unless superklass
        dispatch[object.class] = dispatch[superklass]
        retry
      end
    end
  end
end
{% endhighlight %}

`Hash.new {|hash, key| ...}` はハッシュのデフォルト値を定めます。  
ref. <http://www.ruby-lang.org/ja/old-man/html/Hash.html>

上のコードで、`accept`に渡される引数は`Arel::Nodes::SelectStatement`クラスのインスタンスでした。
したがって、`dispatch[object.class]` が実行された場合、得られる値は`"visit_Nodes_SelectStatement"`となります。

`visit`では、`send`メソッドを呼んでいます。`send`は`Object`クラスのメソッドで、
`obj.send(name,args)`は、`obj`の`name`メソッドを`args`を引数として呼びだします。

つまり、`ToSql`クラスの`visit_Nodes_SelectStatement`というメソッドが呼ばれることになります。

ちなみに、`Visitor`クラスのサブクラスには、

-   `DepthFirst`
-   `Dot`
-   `Informix`
-   `OrderClauses`

などがあります。これらのクラスのインスタンスの`visit`メソッドを呼びだしたときも、同様の処理が行われることになります。


`To_Sql#visit_Nodes_SelectStatement`
----------

{% highlight ruby linenos %}
  def visit_Arel_Nodes_SelectStatement o
    [
      (visit(o.with) if o.with),
      o.cores.map { |x| visit_Arel_Nodes_SelectCore x }.join,
      ("ORDER BY #{o.orders.map { |x| visit x }.join(', ')}" unless o.orders.empty?),
      (visit(o.limit) if o.limit),
      (visit(o.offset) if o.offset),
      (visit(o.lock) if o.lock),
    ].compact.join ' '
  end
{% endhighlight %}

`o.cores`の中に、`books.where`に渡された引数が格納されています。

{% highlight ruby linenos %}
  def visit_Arel_Nodes_SelectCore o
    [
      "SELECT",
      (visit(o.top) if o.top),
      (visit(o.set_quantifier) if o.set_quantifier),
      ("#{o.projections.map { |x| visit x }.join ', '}" unless o.projections.empty?),
      ("FROM #{visit(o.source)}" if o.source && !o.source.empty?),
      ("WHERE #{o.wheres.map { |x| visit x }.join ' AND ' }" unless o.wheres.empty?),
      ("GROUP BY #{o.groups.map { |x| visit x }.join ', ' }" unless o.groups.empty?),
      (visit(o.having) if o.having),
    ].compact.join ' '
  end
{% endhighlight %}

ここで察しがつくかと思いますが、ここから`visit`を再帰的に呼び出していきます。

`o.wheres` に `books.where` に渡された引数 `books[:title].eq(...)` が格納されています。
`books[:title].eq(...)`のクラスは`Arel::Nodes::Equality`なので、8行目で`visit_Arel_Nodes_Equality`が呼ばれます。

{% highlight ruby linenos %}
  def visit_Arel_Nodes_Equality o
    right = o.right

    if right.nil?
      "#{visit o.left} IS NULL"
    else
      "#{visit o.left} = #{visit right}"
    end
  end
{% endhighlight %}

`visit o.left`の部分で、`books[:title]`に対して`visit`が呼ばれます。

{% highlight ruby linenos %}
  def visit_Arel_Attributes_Attribute o
    self.last_column = column_for o
    join_name = o.relation.table_alias || o.relation.name
    "#{quote_table_name join_name}.#{quote_column_name o.name}"
  end
{% endhighlight %}

最終的に、`"SELECT FORM books WHERE books.title = 'Head First Rails'"`という文字列が得られるはずです。

まとめ
----------

Arelに特徴的なのは、`#where` や `#eq` というメソッドが呼ばれたときに、
特に具体的な変換は行わず、引数や`self`を保持するインスタンスを返すだけ、ということではないでしょうか。

いってみれば、`#where`や`#eq`などを使いながら「SQL構文木」を構築していくのだと思います。

`#to_sql`が呼ばれたときに、この「構文木」のノードや葉を対応するSQLに変換していくわけです。
