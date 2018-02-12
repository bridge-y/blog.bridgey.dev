---
title: "ブログ周りのツール"
date: 2018-02-12T23:31:47+09:00
draft: true
tags: ["Hugo", "Netlify", "Bitbucket", "IFTTT"]
---

前回はブログで扱う内容と自己紹介でしたので，今回が初めてのまともな記事となります。  
本記事では、本ブログを構成するツールやサービスを記したいと思います。  
具体的な構築手順や設定には触れません。  


ブログ構成
---

{{% img src="images/blog-architecture.png" %}}

上図は記事投稿から公開までの流れを図にしたものです。  
（きれいな図を作れるようになりたい）  

使用しているツール，サービスは以下の通りです。  

- [Hugo](https://gohugo.io/)
- [Bitbucket](https://bitbucket.org/)
- [Netlify](https://www.netlify.com/)
- [IFTTT](https://ifttt.com/)


# Hugo

静的サイトジェネレータ。Markdown形式で記事を書けます。  
静的サイトジェネレータはHugoが初めてなので他のツールと比較したわけではありませんが、使いやすいと思います。  

本ブログのテーマは[Robust](https://github.com/dim0627/hugo_theme_robust)を使っています。  
以下のサイトを参考に少しカスタマイズしています。  

- [HUGO のテーマ Robust のカスタマイズver3 - zzzmisa's blog](http://blog.zzzmisa.com/customize_hugo_theme3/)  
- [Robust設定 :: Alice in the Machine - wiki](https://browniealice.github.io/wiki/technote/hugo/setting_for_robust/)

記事の雛形をHugoで生成して、Vimで文章を書いています。  
記事の推敲を終えたらBitbucketへプッシュ。  


# Bitbucket

Gitホスティングサービス。本ブログリポジトリの管理先です。  
Bitbucketは自分用ツールの管理に重宝してます。プライベートリポジトリが無料なのが便利。  
Bitbucketのアカウントを最初に作ったのでGitHubアカウントは持っていないです。  


# Netlify

ホスティングサービス。本ブログをホスティングしています。  
無料でGitリポジトリ連携、HTTPS化、独自ドメイン設定などの様々な機能を使うことができます。  
Bitbucketへ記事をプッシュしたら、ビルドとホスティングをいっぺんにやってくれます。  

パスワード管理ツールである[keeweb](https://github.com/keeweb/keeweb)をホスティングするために半年くらい前から使い始めました。  


# IFTTT

Webサービス間の連携プラットフォーム。  
条件と処理を設定しておき、条件が満たされたときに自動的に設定した処理を実行してくれます。  
私は数年前からライフログの記録や簡単な通知などに使い倒してます。  

本ブログでは、RSSから新しい記事の投稿を検知し、Twitterに投稿するよう設定しています。  
この記事を作成する前に設定したので、まだちゃんと動くか確かめられていないです。  
（きっと自動的にtweetされたはず……）  


Slackにも通知が飛ぶようにしていますが、こちらはIFTTTではなくSlackのRSSインテグレーションを利用しています。  

やってみたいこと
---

Google Adsenseや独自ドメイン取得はそのうちやってみたいと考えています。  
あと、Hugoのテーマ作成もやってみたいですね。  
HTMLやJavaScriptを始めとしたWeb系の知識はほとんどないので、遊びながら学習していきたいです。  


おわりに
---

使い慣れたツール・サービスが半分、使い始めて日の浅いものが半分といったところです。  
Netlifyは今後別の形でもお世話になる予感がしています。  


本記事では名称の紹介程度でしたが、具体的な設定などはそのうちまた書きたいです。  
（keewebやライフログについても書きたい）


以上、ブログ周りのツール紹介でした。  

