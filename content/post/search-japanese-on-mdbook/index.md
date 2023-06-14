---
title: "mdBookで日本語検索"
slug: search-japanese-on-mdbook
author: ""
type: ""
date: 2023-06-14T07:49:56+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ["mdBook"]
---

[mdBook](https://github.com/rust-lang/mdBook)はデフォルトで日本語検索できない。

調べたら日本語で検索できる方法を見つけたので、備忘録として書いておく。

## 方法

以下の issue に書かれているスクリプトをダウンロードし、`book.toml` に記述する。

[Support CJK (mutiple language) search · Issue #2052 · rust-lang/mdBook](https://github.com/rust-lang/mdBook/issues/2052)

1. スクリプトをダウンロードする。

   ```bash
   mkdir assets
   cd ./assets
   curl https://github.com/HillLiu/docker-mdbook/blob/main/mdbook-demo/assets/fzf.umd.js -O
   curl https://github.com/HillLiu/docker-mdbook/blob/main/mdbook-demo/assets/elasticlunr.js -O
   ```

2. `book.toml` を編集する。

   ```toml
   [output.html]
   additional-js = ["assets/fzf.umd.js", "assets/elasticlunr.js"]
   ```
