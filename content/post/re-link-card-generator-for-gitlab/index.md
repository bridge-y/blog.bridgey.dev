---
title: "続・GitLab のリンクカードジェネレーターを作った"
slug: re-link-card-generator-for-gitlab
author: ""
type: ""
date: 2023-11-20T22:17:45+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ["GitLab", "Hono", "Cloudflare Pages", "TypeScript", "Alpine.js"]
---

以前作った GitLab のリンクカードジェネレーターの Web UI を作った。  
参考：[GitLab のリンクカードジェネレーターを作った]({{<ref "/post/re-link-card-generator-for-gitlab/index.md">}})

Web UI のリンクはこちら → [gitlab-card.pages.dev](https://gitlab-card.pages.dev/)

例によって、[gh-card](https://github.com/nwtgck/gh-card) を参考にしている。

技術スタックは参考にした gh-card から変えていて、[Hono](https://hono.dev/) + [Tailwind CSS](https://tailwindcss.com/) + [Alpine.js](https://alpinejs.dev/) を使用した。
それぞれ参考にしたページは以下の通り。

- [Honoの新しいCloudflare Pagesスターターについて](https://zenn.dev/yusukebe/articles/92fcb0ef03b151)
- [HonoとDenoで社内ツールを作ってみた - RAKSUL TechBlog](https://techblog.raksul.com/entry/2023/11/08/173335)

フロントエンドをよくわかっていないため、どうせなら Rust で実装しようかと思い最初は [Dioxus](https://dioxuslabs.com/) を使って書いていた。

しかし、フロントエンドの知識が乏しい状態では実装がままならなかったため、結局 TypeScript で実装した。

もう少し TypeScript を使ってフロントエンドの知識を獲得してから、また Dioxus での実装にチャレンジしようと思う。
