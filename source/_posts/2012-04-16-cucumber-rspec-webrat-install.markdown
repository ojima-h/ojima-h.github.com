---
layout: post
title: "cucumber + rspec + webrat の落とし穴"
date: 2012-04-16 21:17
comments: true
categories: rails
---

cucumber + rspec + webrat ではまったのでメモ。

visit が効かない
----------------

    When /^visit "(.*)"$/ do |page|
	  visit page
	  p response
	end
	
ってやると、`nil` が表示される。。。


調べていくと、こんなページを発見。

<http://forums.pragprog.com/forums/95/topics/4951>

とりあえず、features/support/env.rb に以下を追加。

    require 'webrat'
	
    Webrat.configure do  |config|
      config.mode = :rack
    end

そしたら、こんなエラーが発生。。。

    No response yet. Request a page first. (Rack::Test::Error)
	
<http://forums.pragprog.com/forums/95/topics/8962>

このページを参考に、step の定義を次のように変更.

    When /^visit "(.*)"$/ do |url|
	  visit url
	  p page
	end
	
無事実行されました。


have_tag にブロックを渡せない
-----------------------------

    page.body.should have_selector('a') do |a|
      p a
    end

ってやっても期待する結果が得られない。。。

未だ解決策みつからず。


まぁ、いいか
