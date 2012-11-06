---
layout: post
title: "Github pages + Octpress"
date: 2012-04-15 13:44
comments: true
categories: octpress
---

GitHub Pages + Octpressでブログ立ち上げるまでの記録。

-   <http://mattn.kaoriya.net/software/lang/ruby/20111017205717.htm>  
-   <http://octopress.org/docs/setup/>  
-   <http://octopress.org/docs/deploying/github/>  
-   <http://octopress.org/docs/blogging/>  

を参考に。


環境は、Gentoo-3.2.1-r2 / ruby 1.9.3p0

1.  _username_.github.com というレポジトリを新規作成

2.  Octpressのソースを取得。

        git clone https://github.com/imathis/octopress
		cd octopress

3.  インストール

        rake install
		rake setup_github_pages
		
	`rake setup_github_pages` するとGitレポジトリのURIを聞かれるので、
	さっき作ったレポジトリ (= `git@github.com/username/username.github.com`) を指定する。
	ここで、`.git/config`を書き換えて、orgin とかを設定してくれるぽい。
	
4.  とりあえず、なにか投稿してみる。

        rake new_post["hello world"]
		
	"hello world"の部分はポストのタイトルを指定します。
	ここで指定したものがURLになります。  
	(実際は、`.../:year/:month/:day/:title`ってかんじ)
	
	ページに表示されるタイトルはあとからいじれます。
	
	実行すると、`source/_posts/<yaer>-<month>-<day>-hello-world.markdown`というファイルが
	生成されます。
	
		---
		layout: post
		title: こんにちは
		date: 2012-04-14 23:31
		comments: true
		categories: diary
		---
		
		こんにちは! 世界!
		------------------
		
		むにゃむにゃ
		
	こんなかんじて適当に編集します。
	
5.  さあ、公開してみましょう

    今作成したファイルをデプロイ用のディレクトリにコピーします。

        rake generate

    この段階では、まだアップロードされてません。
	
	アップするまえにローカル環境で確認することができます。

        rake preview
		
	として、`localhost:4000`にアクセスすればおけ。
	
	問題がないようなら、githubにあげます。
	
	    rake deploy
		
6.  成功していれば (失敗しても)、githubからメールが来ます。


ちなみに `rake deploy`でpushされるのは、public/ 以下の内容のみ。
ソース全体もpushしておきましょう。(って書いてあった)

    git add .
	git commit -m "message"
	git push origin source

----------------

なにが便利かっていうと

-   手元で編集できる。Emacsとかでね。
-   localhost でプレビューできる。
-   rake 一発で公開できる。


そういえば、Octpressみたいな形で配布されるアプリ(？)って初めて見た。  
たしかに、Rakefile に色々書いておけばシステムにインストールする必要ないしな。。。

それにしても便利。


		
		
			
