---
layout: post
title: "Hive クエリを最適化する"
date: 2013-08-30 11:23
comments: true
categories: 
---

- [Hadoop本](http://www.amazon.co.jp/Hadoop-Tom-White/dp/487311439X)
- [HADOOP HACKS](http://www.amazon.co.jp/Hadoop-Hacks-―プロフェッショナルが使う実践テクニック-中野-猛/dp/4873115469)

を参考に、HiveQL が どんな Map/Reduce タスクに展開されるのかを想像しつつ(ソースは読んでないのであくまで想像)、
効率の良い Hiveクエリの書き方を考えてみる。

まずは、普通のクエリ
----------

    SELECT * FROM movie

は、どんな Map/Reduce タスクに変換されるんでしょうか？


hive で

    > EXPLAIN SELECT * FROM movie;

とやってみると、

    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_TABREF (TOK_TABNAME movie))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_SELEXPR TOK_ALLCOLREF))))
     
    STAGE DEPENDENCIES:
      Stage-0 is a root stage
     
    STAGE PLANS:
      Stage: Stage-0
        Fetch Operator
          limit: -1

と返っくる。
たぶん、`Stage-0` は「ただ吐き出す」ていうタスクなんだろう。


`SELECT * FROM movie LIMIT 10` と比較してみる:

    > EXPLAIN SELECT * FROM movie LIMIT 10;

    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_TABREF (TOK_TABNAME all_member))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_SELEXPR TOK_ALLCOLREF)) (TOK_LIMIT 10)))
    
    STAGE DEPENDENCIES:
      Stage-0 is a root stage
    
    STAGE PLANS:
      Stage: Stage-0
        Fetch Operator
          limit: 10
        

`Stage-0` は、`LIMIT` の面倒だけ見ている気がする。



SELECT WHERE してみる
-----------------

    > EXPLAIN SELECT id FROM movie WHERE year = 2000 LIMIT 10;

    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_TABREF (TOK_TABNAME movie))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_SELEXPR (TOK_TABLE_OR_COL id))) (TOK_WHERE (= (TOK_TABLE_OR_COL year) 2000))))
     
    STAGE DEPENDENCIES:
      Stage-1 is a root stage
      Stage-0 is a root stage
     
    STAGE PLANS:
      Stage: Stage-1
        Map Reduce
          Alias -> Map Operator Tree:
            movie 
              TableScan
                alias: movie
                Filter Operator
                  predicate:
                      expr: (year = 2000)
                      type: boolean
                  Select Operator
                    expressions:
                          expr: id
                          type: int
                    outputColumnNames: _col0
                    File Output Operator
                      compressed: false
                      GlobalTableId: 0
                      table:
                          input format: org.apache.hadoop.mapred.TextInputFormat
                          output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
     
      Stage: Stage-0
        Fetch Operator
          limit: -1


たぶん mapper はこんなイメージかな...？

    public void map(Integer key, Text row, OutputCollector<Integer, Text> output, ...) {
        Integer id = row.getId();
        Integer year = row.getYear();
        
        if (year == 2000) {
            output.collect(key, new Writable(id));
        }
    }


GROUP BY してみる
-------------

    > EXPLAIN SELECT year, count(*) FROM movie GROUP BY year;
    
    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_TABREF (TOK_TABNAME movie))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_SELEXPR (TOK_TABLE_OR_COL year)) (TOK_SELEXPR (TOK_FUNCTIONSTAR count))) (TOK_GROUPBY (TOK_TABLE_OR_COL year))))
    
    STAGE DEPENDENCIES:
      Stage-1 is a root stage
      Stage-0 is a root stage
    
    STAGE PLANS:
      Stage: Stage-1
        Map Reduce
          Alias -> Map Operator Tree:
            movie 
              TableScan
                alias: movie
                Select Operator
                  expressions:
                        expr: year
                        type: string
                  outputColumnNames: year
                  Group By Operator
                    aggregations:
                          expr: count()
                    bucketGroup: false
                    keys:
                          expr: year
                          type: string
                    mode: hash
                    outputColumnNames: _col0, _col1
                    Reduce Output Operator
                      key expressions:
                            expr: _col0
                            type: string
                      sort order: +
                      Map-reduce partition columns:
                            expr: _col0
                            type: string
                      tag: -1
                      value expressions:
                            expr: _col1
                            type: bigint
          Reduce Operator Tree:
            Group By Operator
              aggregations:
                    expr: count(VALUE._col0)
              bucketGroup: false
              keys:
                    expr: KEY._col0
                    type: string
              mode: mergepartial
              outputColumnNames: _col0, _col1
              Select Operator
                expressions:
                      expr: _col0
                      type: string
                      expr: _col1
                      type: bigint
                outputColumnNames: _col0, _col1
                File Output Operator
                  compressed: false
                  GlobalTableId: 0
                  table:
                      input format: org.apache.hadoop.mapred.TextInputFormat
                      output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
    
      Stage: Stage-0
        Fetch Operator
          limit: -1
    
Map Operator Tree の中に Reduce Output Operator があるのは、map task と reduce task 間のデータ転送量を小さくするために
map task 側で集約関数(combiner)を呼ぶためだと思われる。

たぶん mapper はこんなイメージ...

    public void map(Integer key, Text row, OutputCollector<Integer, Text> output, ...) {
        Integer year = row.getYear();
        
        output.collect(year, new Writable(key));
    }


reducer (と combiner) は多分こんな感じ...

    publick void reduce(Text year, Iterator<Text> values, OutputCollector<Text,BigInt> output, ...) {
        BigInt count = values.count();

        output.collect(<何かしらのKEY>, new Writable(new Row(year, count)));
    }


問題は JOIN
--------

    > EXPLAIN SELECT a.id, b.rating FROM movie a JOIN score b on a.id = b.movie_id

    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_JOIN (TOK_TABREF (TOK_TABNAME movie) a) (TOK_TABREF (TOK_TABNAME score) b) (= (. (TOK_TABLE_OR_COL a) id) (. (TOK_TABLE_OR_COL b) movie_id)))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_SELEXPR (. (TOK_TABLE_OR_COL a) id)) (TOK_SELEXPR (. (TOK_TABLE_OR_COL b) rating)))))
    
    STAGE DEPENDENCIES:
      Stage-1 is a root stage
      Stage-0 is a root stage
    
    STAGE PLANS:
      Stage: Stage-1
        Map Reduce
          Alias -> Map Operator Tree:
            a 
              TableScan
                alias: a
                Reduce Output Operator
                  key expressions:
                        expr: id
                        type: int
                  sort order: +
                  Map-reduce partition columns:
                        expr: id
                        type: int
                  tag: 0
                  value expressions:
                        expr: id
                        type: int
            b 
              TableScan
                alias: b
                Reduce Output Operator
                  key expressions:
                        expr: movie_id
                        type: int
                  sort order: +
                  Map-reduce partition columns:
                        expr: movie_id
                        type: int
                  tag: 1
                  value expressions:
                        expr: rating
                        type: int
          Reduce Operator Tree:
            Join Operator
              condition map:
                   Inner Join 0 to 1
              condition expressions:
                0 {VALUE._col0}
                1 {VALUE._col1}
              handleSkewJoin: false
              outputColumnNames: _col0, _col3
              Select Operator
                expressions:
                      expr: _col0
                      type: int
                      expr: _col3
                      type: int
                outputColumnNames: _col0, _col1
                File Output Operator
                  compressed: false
                  GlobalTableId: 0
                  table:
                      input format: org.apache.hadoop.mapred.TextInputFormat
                      output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
    
      Stage: Stage-0
        Fetch Operator
          limit: -1


処理の流れはたぶんこんな感じ:

1.  __MAPPER__

    `movie` テーブルと `score` テーブルから必要なカラムを抜きだす。

2.  __REDUCER__

    mapper からデータをコピーする。このとき、同じ結合キー (movie.id と score.movie_id) を持つデータは
    同じ reduce task にコピーされる。
    で、結合して、指定されたカラムを返す。


MAPJOIN してみる
------------

HADOOP HACKS とかに、マップサイドジョンが紹介されていたので、試してみる。

    > EXPLAIN SELECT /*+ MAPJOIN(a) */ id, rating FROM movie a JOIN score b ON a.id = b.movie_id;
    
    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_JOIN (TOK_TABREF (TOK_TABNAME movie) a) (TOK_TABREF (TOK_TABNAME score) b) (= (. (TOK_TABLE_OR_CO
    L a) id) (. (TOK_TABLE_OR_COL b) movie_id)))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_HINTLIST (TOK_HINT TO
    K_MAPJOIN (TOK_HINTARGLIST a))) (TOK_SELEXPR (TOK_TABLE_OR_COL id)) (TOK_SELEXPR (TOK_TABLE_OR_COL rating)))))
    
    STAGE DEPENDENCIES:
      Stage-3 is a root stage
      Stage-1 depends on stages: Stage-3
      Stage-0 is a root stage
    
    STAGE PLANS:
      Stage: Stage-3
        Map Reduce Local Work
          Alias -> Map Local Tables:
            a 
              Fetch Operator
                limit: -1
          Alias -> Map Local Operator Tree:
            a 
              TableScan
                alias: a
                HashTable Sink Operator
                  condition expressions:
                    0 {id}
                    1 {rating}
                  handleSkewJoin: false
                  keys:
                    0 [Column[id]]
                    1 [Column[movie_id]]
                  Position of Big Table: 1
    
      Stage: Stage-1
        Map Reduce
          Alias -> Map Operator Tree:
            b 
              TableScan
                alias: b
                Map Join Operator
                  condition map:
                       Inner Join 0 to 1
                  condition expressions:
                    0 {id}
                    1 {rating}
                  handleSkewJoin: false
                  keys:
                    0 [Column[id]]
                    1 [Column[movie_id]]
                  outputColumnNames: _col0, _col3
                  Position of Big Table: 1
                  Select Operator
                    expressions:
                          expr: _col0
                          type: int
                          expr: _col3
                          type: int
                    outputColumnNames: _col0, _col3
                    Select Operator
                      expressions:
                            expr: _col0
                            type: int
                            expr: _col3
                            type: int
                      outputColumnNames: _col0, _col1
                      File Output Operator
                        compressed: false
                        GlobalTableId: 0
                        table:
                            input format: org.apache.hadoop.mapred.TextInputFormat
                            output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
          Local Work:
            Map Reduce Local Work
    
      Stage: Stage-0
        Fetch Operator
          limit: -1
    
たしかに reduce task がなくなっている。

<https://cwiki.apache.org/confluence/display/Hive/LanguageManual+JoinOptimization> には、MAPJOIN の説明が以下のようにされている。

>   MAPJOINs are processed by loading the smaller table into an in-memory hash map and matching keys with the larger table as they are streamed through. The prior implementation has this division of labor:
> 
>   -   Local work:
>       -   read records via standard table scan (including filters and projections) from source on local machine
>       -   build hashtable in memory
>       -   write hashtable to local disk
>       -   upload hashtable to dfs
>       -   add hashtable to distributed cache
>   -   Map task
>       -   read hashtable from local disk (distributed cache) into memory
>       -   match records' keys against hashtable
>       -   combine matches and write to output
>   -   No reduce task

Stage-3 で、`movie` テーブルを読み込んで hashtable を構築しているようだ。
で、Stage-1 の map task で、この hashtable を使いながら `socre` テーブルを読み込んでいくのだろう。


Map Side Join についてもう少し考えてみる
---------------------------

Hadoop本の中で、「map側結合」が紹介されていた。(p.254 8.3.1 map側結合)
ここで説明されている map 側結合と、Hive の MAPJOIN はどうも違うようだ。

Hadoop本の map 側結合が Hive クエリ実行時に行なわれることはないのだろうか？
Join 対象になる2つのテーブル (もしくは、サブクエリの結果) のデータが、
結合キーごとに同じデータノードに入っている状況。

うん。。

思いつかない。


その他の JOIN の最適化方法
----------------

- <https://cwiki.apache.org/confluence/download/attachments/27362054/Hive+Summit+2011-join.pdf?version=1&modificationDate=1310001042000>
- <https://cwiki.apache.org/confluence/display/Hive/LanguageManual+JoinOptimization>
- <https://cwiki.apache.org/confluence/display/Hive/Skewed+Join+Optimization>

とかもあるみたい。

### Bucket Map Join

Join 対象のテーブルが bucketize されている場合、構築された hashtable を bucket key に基づいて、必要な map task にだけ配布するっぽい。

### Sort Merge Bucket Map Join

Join 対象のテーブルが bucketize されていて、しかも sort されている場合、もっと早くなるみたい。

でも、sort するのが大変。

試してみた。

    > set hive.input.format=org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
    > set hive.optimize.bucketmapjoin = true;
    > set hive.optimize.bucketmapjoin.sortedmerge = true;
    > explain select  /*+ mapjoin(b) */ id, rating from movie a join score b on a.id = b.movie_id;

    ABSTRACT SYNTAX TREE:
      (TOK_QUERY (TOK_FROM (TOK_JOIN (TOK_TABREF (TOK_TABNAME movie) a) (TOK_TABREF (TOK_TABNAME score) b) (= (. (TOK_TABLE_OR_COL a) id) (. (TOK_TABLE_OR_COL b) movie_id)))) (TOK_INSERT (TOK_DESTINATION (TOK_DIR TOK_TMP_FILE)) (TOK_SELECT (TOK_HINTLIST (TOK_HINT TOK_MAPJOIN (TOK_HINTARGLIST b))) (TOK_SELEXPR (TOK_TABLE_OR_COL id)) (TOK_SELEXPR (TOK_TABLE_OR_COL rating)))))
    
    STAGE DEPENDENCIES:
      Stage-1 is a root stage
      Stage-0 is a root stage
    
    STAGE PLANS:
      Stage: Stage-1
        Map Reduce
          Alias -> Map Operator Tree:
            a 
              TableScan
                alias: a
                Sorted Merge Bucket Map Join Operator
                  condition map:
                       Inner Join 0 to 1
                  condition expressions:
                    0 {id}
                    1 {rating}
                  handleSkewJoin: false
                  keys:
                    0 [Column[id]]
                    1 [Column[movie_id]]
                  outputColumnNames: _col0, _col7
                  Position of Big Table: 0
                  Select Operator
                    expressions:
                          expr: _col0
                          type: int
                          expr: _col7
                          type: int
                    outputColumnNames: _col0, _col7
                    Select Operator
                      expressions:
                            expr: _col0
                            type: int
                            expr: _col7
                            type: int
                      outputColumnNames: _col0, _col1
                      File Output Operator
                        compressed: false
                        GlobalTableId: 0
                        table:
                            input format: org.apache.hadoop.mapred.TextInputFormat
                            output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
    
      Stage: Stage-0
        Fetch Operator
          limit: -1

**すごい！**
**reduce task が無い上に、hashtable の構築すら消えた！！**

これが Hadoop 本で言っていた map側結合なのかもしれない。
    

### Skewed Join

よく分からない。



その他、思ったこと
---------

- MAPJOIN はテーブル1つをまるごとメモリに載せないといけないので、
  せいぜい数十メガから数百メガのテーブルじゃないと辛そう。
  サブクエリの中で十分絞り込んだ後 JOIN するのが実用的かも。

- Bucket Map Join は、既存のHiveテーブルに対して、後から bucketize するのは大変そう。
  集計用の中間テーブルを作ることがあったら、検討してみるといいかも。

- Semi Join を使うのも有効らしい

- HADOOP HACKS の中で、UDF (User Define Fucntion) や UDAF (User Define Aggregate Function) の作り方が載っていた。
  この辺をもっとサクサク使えるようになるとはかどりそうな気がした。
  むしろ、使わないともったいない。
