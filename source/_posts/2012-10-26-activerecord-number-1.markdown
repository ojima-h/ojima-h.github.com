---
layout: post
title: "ActiveRecord #1"
date: 2012-10-26 08:40
comments: true
categories: rails-sourcecode-reading inov2013
---

前回 Arel が SQL を構築するところを読んだので、次は ActiveRecord がデータベースを操作するときに、どのタイミングで SQL を構築しているのかを見たいと思います。

{% highlight ruby linenos %}
require 'debugger'
require 'active_record'

ActiveRecord::Base.configurations = {
  'development' => {
    :adapter => 'sqlite3',
    :database => './sampledb.sqlite'
  }
}
ActiveRecord::Base.establish_connection('development')

class User < ActiveRecord::Base; end

p User.where(:name => "John").limit(10)
{% endhighlight %}

このサンプルで、`User.where(...)`の時点でSQLが発行されてしまうと、その後に続く`.limit(10)`が反映されないはずじゃないか、というのか今回の疑問です。

* * * * *

まず、`#where`の場所を探します。

`ActiveRecord`はざっくりこんなかんじになっています↓

{% highlight ruby %}
module ActiveRecord
  class Base
    extend Querying
    include Scoping
  end
  
  module Querying
    delegate :select, :group, :order, :except, :reorder, :limit, :offset, :joins,
             :where, :preload, :eager_load, :includes, :from, :lock, :readonly,
             :having, :create_with, :uniq, :to => :scoped
  end            
  
  module Scoping
    included do
      include Named
    end

    module Named
      def scoped(options = nil)
        if options
          scoped.apply_finder_options(options)
        else
          if current_scope
            current_scope.clone
          else
            scope = relation.clone
            scope.default_scoped = true
            scope
          end
        end
      end
    end
  end

  class Relation
    include QueryMethods, Delegation
  end

  module QueryMethods
    def where(opts, *rest)
      return self if opts.blank?
   
      relation = clone
      relation.where_values += build_where(opts, rest)
      relation
    end
  end
end
{% endhighlight %}

Module#delegate
----------

ref. <http://api.rubyonrails.org/classes/Module.html#method-i-delegate>

`delegate` は、`active_support`の中で`Module#delegate`として定義されています。

{% highlight ruby %}
class Hoge
  delegate method, :to => target

  def target
    ...
  end
end
{% endhighlight %}

上の例では、`Hoge#method`の呼び出すと、`Hoge#target`が返すオブジェクトの`method`メソッドが呼ばれます。

`Concern#included`
----------

`included`メソッドも同じく`active_support`で定義されており、
そのモジュールがincludeされた時に、ブロックの中身が実行されます。

----

というわけで、`Relation#where`が呼び出されることが分かります。

`Relation#where`
----------

`build_where`の中身を追っていくと、
`PredicateBuilder#build_from_hash`内の以下のコードが実行されることが分かります。

{% highlight ruby %}
  def self.build_from_hash(engine, attributes, default_table, allow_table_name = true)
    predicates = attributes.map do |column, value|
      table = default_table
      
      ....

      attribute = table[column.to_sym]

      case value
      when ActiveRecord::Relation
        ...
      when Array, ActiveRecord::Associations::CollectionProxy
        ...
      ...
      else
        attribute.eq(value)
      end
    end
    predicates.flatten
  end
{% endhighlight %}

上のコード内の`default_table`には
`Arel::Table`のインスタンスが入っています。

とういわけで、`User.where(:name => "John")`を実行したとき、
`Relation#build_where`の結果は`[table[:name].eq("John")]`となります。
これが`Relation#where_values`に格納されます。

以上で、`ActiveRecord::Base#where`の呼び出しは終わりです。

まとめると、`ActiveRecord::Base#where`を呼び出すと、`Relation`クラスのインスタンスが返されます。
その中には`table[column].eq(value)`たちが入った配列が格納されています。

`Relation#limit`
----------

`Relation#limit`も同じかんじです。

{% highlight ruby linenos %}
  def limit(value)
    relation = clone
    relation.limit_value = value
    relation
  end
{% endhighlight %}

で、いつSQL？
----------

さて、`User.where(..).limit(10)`の結果、`Relation`クラスのインスタンスが返ってくることは分かりましたが、
これがいつSQLに変換され、データベースに向けて投げられるんでしょうか？

`where`の検索結果が必要になるのはどんな時でしょうか？

大抵の場合、検索結果から個々の`User`クラスのインスタンスを取り出すときかと思います。
このとき、検索結果に対しては Array としてアクセスすると思います。

というわけで、`Relation#to_a`について見てみました。


{% highlight ruby linenos %}
  def to_a
    logging_query_plan do
      exec_queries
    end
  end
  def exec_queries
    return @records if loaded?

    default_scoped = with_default_scope

    if default_scoped.equal?(self)
      @records = if @readonly_value.nil? && !@klass.locking_enabled?
        eager_loading? ? find_with_associations : @klass.find_by_sql(arel, @bind_values)
      else
        IdentityMap.without do
          eager_loading? ? find_with_associations : @klass.find_by_sql(arel, @bind_values)
        end
      end
    ...
  end
{% endhighlight %}

`scope`とか`eager_loading`とかよく分からないことは置いておいて、デバッガで追いかけると、
`@klass.find_by_sql(arel, @bind_values)`が実行されることが分かります。

ということは、`arel`の値が問題になりそうです。


{% highlight ruby linenos %}
module ActiveRecord
  module QueryMethods
    ...
    def arel
      @arel ||= with_default_scope.build_arel
    end

    def build_arel
      arel = table.from table

      build_joins(arel, @joins_values) unless @joins_values.empty?

      collapse_wheres(arel, (@where_values - ['']).uniq)

      arel.having(*@having_values.uniq.reject{|h| h.blank?}) unless @having_values.empty?

      arel.take(connection.sanitize_limit(@limit_value)) if @limit_value
      arel.skip(@offset_value.to_i) if @offset_value

      arel.group(*@group_values.uniq.reject{|g| g.blank?}) unless @group_values.empty?

      order = @order_values
      order = reverse_sql_order(order) if @reverse_order_value
      arel.order(*order.uniq.reject{|o| o.blank?}) unless order.empty?

      build_select(arel, @select_values.uniq)

      arel.distinct(@uniq_value)
      arel.from(@from_value) if @from_value
      arel.lock(@lock_value) if @lock_value

      arel
    end
    def collapse_wheres(arel, wheres)
      equalities = wheres.grep(Arel::Nodes::Equality)

      arel.where(Arel::Nodes::And.new(equalities)) unless equalities.empty?

      (wheres - equalities).each do |where|
        where = Arel.sql(where) if String === where
        arel.where(Arel::Nodes::Grouping.new(where))
      end
    end
{% endhighlight %}

こんなかんじになってました。
この`build_arel`で、`where`とか`limit`に渡された引数たちを、
まとめて`Arel`のオブジェクトに変換していってます。

`arel.xxx`が連なってる感じ、壮観です。

`find_by_sql`
----------

この時点では`arel`はSQLになっていません。

なので、`find_by_sql`の中を追っていくことにします。

追っていくと、
`ActiveRecord::ConnectionAdapters::DatabaseStatements#select_all`
に行き着きます。

{% highlight ruby linenos %}
module ActiveRecord
  module ConnectionAdapters # :nodoc:
    module DatabaseStatements
      # Converts an arel AST to SQL
      def to_sql(arel, binds = [])
        if arel.respond_to?(:ast)
          visitor.accept(arel.ast) do
            quote(*binds.shift.reverse)
          end
        else
          arel
        end
      end

      # Returns an array of record hashes with the column names as keys and
      # column values as values.
      def select_all(arel, name = nil, binds = [])
        select(to_sql(arel, binds), name, binds)
      end
{% endhighlight %}

`DatabaseManager#to_sql`の中の
`visitor.accept(arel.ast) do ...` というのが、
`Arel::TreeManger#to_sql`と同じことをやっているのが分かります。

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

というわけで、ここに来て、Arelのオブジェクトが無事SQLに変換されて、
DBMに向けて投げられたことが分かりました。

めでたし。

`to_a`は誰が呼ぶ？
----------

`to_a`を呼び出せば、SQLが発行されることは分かりましたが、
誰がどのタイミングで`to_a`を呼ぶのでしょうか？

そういえば、`where`の検索結果に対しては、`each`メソッドを呼べます。

{% highlight ruby linenos %}
User.where(:name => "John").each do ...
{% endhighlight %}

`each`がどこで定義されているのか調べてみました。

{% highlight ruby linenos %}
class Relation
  include FinderMethods, Calculations, SpawnMethods, QueryMethods, Batches, Explain, Delegation
  ...
end
module Delegation
  # Set up common delegations for performance (avoids method_missing)
  delegate :to_xml, :to_yaml, :length, :collect, :map, :each, :all?, :include?, :to_ary, :to => :to_a
  delegate :table_name, :quoted_table_name, :primary_key, :quoted_primary_key,
           :connection, :columns_hash, :auto_explain_threshold_in_seconds, :to => :klass
  ...
end
{% endhighlight %}

なるほど。
`each`とか`map`などが呼ばれたタイミングで、`to_a`にデリゲートされると。

`Relation#inspect`
----------

では、たとえば、irbの中で、`> p User.where(...)`などとしたときはどうなんでしょうか？

このときも`to_a`が呼ばれるのでしょうか？

<http://doc.ruby-lang.org/ja/1.9.2/class/Object.html>によると↓とのこ
とです。

>   inspect -> String
>    
>       オブジェクトを人間が読める形式に変換した文字列を返します。
>    
>       組み込み関数 Kernel.#p は、このメソッドの結果を使用して オブジェクトを表示します。

`p`は`inspect`の結果を表示するようです。

`Relation#insepct`は

{% highlight ruby linenos %}
    def inspect
      to_a.inspect
    end
{% endhighlight %}

となってます。

やっぱり、`to_a`が呼ばれるようです。

まとめ
----------

Arelのときもそうでしたが、
`where`などのメソッドが呼ばれた時点では単に引数だけを保存しておいて、
必要になった時点でSQLを発行するということでした。

「必要になった時点」の判断の仕方が面白いと思いました。

必要になった時点で`to_a`を呼ぶわけですが、`each`から`to_a`にdelegateされるとか、
`inspect`の中で`to_a`を呼ぶとか。


しかし、`where`とかをわざわざActiveRecordでラップする必要があるんですかね...
`Relation`のなかでArelのオブジェクトを保持しておいて、`where`の処理はArelにまかせて、
`Relation`は`to_a`とかのまわりだけ面倒みたらだめなのかな...？
