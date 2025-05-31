---
title: "PocketからRaindrop.ioに移行した"
slug: raindrop-io-migration
author: ""
type: ""
date: 2025-06-01T06:17:55+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["Pocket", "Raindrop.io", "自動化", "ライフログ", "IFTTT"]
---

長年愛用してきた Pocket のサービス終了が発表された。  
いくつか移行先を調査し、最終的に[Raindrop.io](https://raindrop.io/)を使うことに決めた。  
移行の経緯と Raindrop.io の運用について紹介する。

## Pocketの活用

Pocket は主に有用だと感じた Web サイトの保存に利用していた。  
特に、[IFTTT](https://ifttt.com/)と連携させることで、Pocket に保存した記事の URL を自動的に[Day One](https://dayoneapp.com/)に記録するようにしていた[^1]。

[^1]: [{{<ref "/post/ifttt-ai-summarizer/index.md">}}]({{<ref "/post/ifttt-ai-summarizer/index.md">}})

## 移行先候補の検討

Web や SNS を調査し、Pocket の移行先として以下の 2 つのサービスに候補を絞った。

- [Raindrop.io](https://raindrop.io/)
- [Readwise](https://readwise.io/)

## 両サービスを試して感じたこと

それぞれのサービスを使ってみて、いくつか特徴と優位点が見えてきた。

### Readwiseの優れていた点

- **読んだ位置の保存**: 記事を途中で閉じた場合でも、次に開いた際に中断した位置から読書を再開できる機能は非常に便利だった。
- **Androidアプリでの保存場所変更**: Android アプリで記事を保存する際、画面遷移なしで保存先を変更できる UI はスムーズでストレスがなかった。
- **ビューアーの表示速度**: 記事ビューアーの表示が速く、サクサクと読み進められる点は好印象だった。
- **ブラウザ拡張でのハイライト作成**: ブラウザ拡張機能を使えば、Web ページ上の重要な部分を簡単にハイライトできた。

### Raindrop.ioの優れていた点

- **コストパフォーマンス**: Android アプリから課金すると年額 2,920 円と、Readwise と比較して費用を抑えることができる。
- **IFTTT連携**: IFTTT に対応しており、自動化が簡単だった。
- **テキスト保存機能**: 課金することで、Web ページのテキスト内容を保存してくれる。リンク切れや元サイトの変更に左右されずに情報を保持できる。
- **タグのレコメンド機能**: Web サイトの内容に応じて関連性の高いタグを自動で提案してくれる。試してみて、非常に体験がよかった。

## Raindrop.ioを選んだ理由

最終的に、以下の点で Raindrop.io を移行先に選んだ。

- **既存運用との親和性**: IFTTT 連携が可能なため、Pocket で構築していた Web サイトのライフログ運用をそのまま継続できる点が最も重要だった。
- **経済性**: 年間コストが Readwise よりも抑えられる点は、長期的な利用を考えると大きなメリットだった。
- **タグレコメンドの体験**: 精度の高いタグレコメンド機能は、手動でのタグ付けの手間を省き、より効率的に情報管理できた。

## 現在のRaindrop.io運用体制

Android で Web サイトを保存する際、Readwise よりも時間がかかる点を改善するために、Web API サーバーを作った。

具体的には、URL を受け取った際に即座にレスポンスを返し、バックグラウンドで以下の処理を実行するようにしている。

- Web ページの本文を要約
- Raindrop.io のレコメンと API からタグを取得
- 取得したタグと要約を付与して Raindrop.io に登録

### PCでの利用

Firefox の拡張機能 [Send Tab URL](https://addons.mozilla.org/en-US/firefox/addon/send-tab-url/)を使って現在開いているタブの URL を自作の API サーバーに URL を送信している。

### Androidでの利用

スマートフォンからは[HTTP Shortcuts](https://http-shortcuts.rmy.ch/)アプリを利用して、同様に API サーバーへ URL を送信し、Raindrop.io へ登録している。

