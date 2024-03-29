---
title: "近況報告（2022年11月）"
slug: recent-report-november-2022
author: ""
type: ""
date: 2022-11-27T08:53:38+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["雑記"]
---

しばらくブログを更新していなかったので、最近やっていたことを簡単にまとめておく。

##  DoHなDNSサーバをホスト

自宅ネットワークではDNSサーバを立てて広告をブロックしている。

これを外出中でも使うために、DoHで自宅ネットワーク外からアクセスできるようにした。

## クリーンアーキテクチャの実践

今までちゃんとプログラム設計について学習していなかったので、たまたまお勧めしているブログで知った[Clean Architecture 達人に学ぶソフトウェアの構造と設計](https://www.kadokawa.co.jp/product/301806000678/)を買って読んだ。

さまざまなケースで設計のポイントが書かれており、得られたものは多かった。

読み終えたあとすぐにクリーンアーキテクチャでプログラムを作ってみたが、わかったつもりになっている点が多く、読むだけでは身につかないことを再認識した。やはり手を動かすことは重要。

読み終えてから3つほどプログラムを作ったので、少しは考え方が身についたように思う。

## TypeScriptの学習

ちょっとフロントエンドを触ってみよう、という気になったので学習を始めた。  
JavaScriptの知識がなくてもなんとかなるだろう、と最初からTypeScriptを使うことにした。  

Denoを使いたかったので、[Freshフレームワーク](https://fresh.deno.dev/)でWebアプリを作った。

ただ、構文の知識がないと技術系のブログやOSSのソースコードを見てもさっぱりわからなかったので、[TypeScriptとReact/Next.jsでつくる 実践Webアプリケーション開発](https://gihyo.jp/book/2022/978-4-297-12916-3)を読みながら進めた。

書籍はReactに関する内容だが、PreactベースであるFreshの参考になる点は多く、たいへん役に立った。

Webのチュートリアルよりも書籍を読んだほうが頭に入る。

まだまだ覚えることは多いので、次に何か作るときも時間がかかりそう。
あと、デザイン難しい。

## 回線速度の可視化

最近自宅の光回線が不調だったので回線速度を可視化することにした。

{{< tweet user="bridgement" id="1573815240147419137" >}}

9月の連休は酷かったが、10月に入ったころにはかなり改善していた。

ちなみに、いまは別の回線を使っている。

不調な期間が長かったから解約手続きをすると決意していたんだ。

## おわりに

本当は10月頭にこのエントリを書き終えるつもりだったのに、あっという間に11月になっていた。

上に書いた以外に認証・認可の実装とかもやっていたが、体調崩してしばらく中断したせいで少し熱が冷めてしまっている。

またモチベーションが上がったら再開したい。
