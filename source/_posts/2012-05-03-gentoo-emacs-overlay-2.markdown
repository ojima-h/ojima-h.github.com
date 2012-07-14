---
layout: post
title: "emacs-overlay で Elisp を管理 -- その2 (プロキシ越しにsvnプロトコル)"
date: 2012-05-03 02:55
comments: true
categories: [gentoo, portage, emacs, elisp]
---

大学のプロキシ内からだと svn プロトコルが通らず、emacs-overlay を layman に追加できなかった。

それを無理矢理なんとかするまでの記録。


1.  `layman -a emacs` するもタイムアウトのエラーが発生。

    `/usr/bin/svn co svn://overlays.gentoo.org/proj/emacs/emacs-overlay/@ /var/lib/layman/emacs` でコケるようだ。
	
2.  直接、`/usr/bin/svn co svn://overlays.gentoo.org/proj/emacs/emacs-overlay/@ emacs` を実行してみるも、やはりムリ。

3.  ならばと、`/usr/bin/svn co https://overlays.gentoo.org/proj/emacs/emacs-overlay/@ emacs` を実行してみるも、やはりムリ。

        svn: パス 'https://overlays.gentoo.org/proj/emacs/emacs-overlay' が見つかりません
		
    とか言われる。
	
4.  困る。

    SSHトンネル掘ってナントカ。。。とかヨクワカラナイコトを考えた挙句、VPS に emacs-overlay のクローンを置いて http でアクセスすることにする。
    ちなみに、VPS の方は Debian です。
	
VPS に emacs-overlay のクローンを置いて http でアクセス
------------

Apache はセットアップ済みで、公開ディレクトリは /var/www であるものとします。

1.  git-svn をインストール

        aptitude install git-svn
		
2.  emacs-overlay をコピーしてくる。

        git svn clone svn://overlays.gentoo.org/proj/emacs/emacs-overlay/ emacs-overlay
		
3.  公開ディレクトリに bare レポジトリを置く。

        cd /var/www/
		git clone /path/to/emacs-overlay emacs-overlay.git --bare

これで、`git clone http://xxxxxx/emacs-overlay.git` でクローンできるようになっているはず。


layman の overlay のリストに追加する
----------------

1.  /var/lib/layman/my-list.xml を作成して、次のような内容を書く。

        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE repositories SYSTEM "/dtd/repositories.dtd">
        <repositories xmlns="" version="1.0">
          <repo quality="experimental" status="official">
            <name><![CDATA[emacs-clone]]></name>
            <description><![CDATA[Provide Emacs related ebuilds which are a bit experimental or
            work-in-progress. Don't rely on them, but don't hesitate to file bugs or
            write emails.]]></description>
            <homepage>http://your/homepage/uri</homepage>
            <owner type="project">
              <email>sample@example.com</email>
            </owner>
            <source type="git">http://server/uri/emacs-overlay.git</source>
          </repo>
        </repositories>
		
2.  /etc/layman/layman.cfg に次のような内容を書く:

        .....
		
        overlays  : http://www.gentoo.org/proj/en/overlays/repositories.xml
                    file:///var/lib/layman/my-list.xml
					
		.....
			

3.  layman を更新する。

        layman -S
		
4.  `layman -L` を実行して、いま追加したオーバレイが表示されれば OK.



----------

振り返ってみると、がんばったわりに得るものが少ない気が。。。


あとは、更新が面倒くさそう。Git の bare レポジトリって直接更新できないのかな...
