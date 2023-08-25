---
title: "GitLab のリンクカードジェネレーターを作った"
slug: link-card-generator-for-gitlab
author: ""
type: ""
date: 2023-08-24T22:58:48+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["GitLab", "Cloudflare Workers", "Rust"]
---

ブログに GitLab のプロジェクトをリンクカードで載せようとして、 GitLab 向けのリンクカードジェネレーターがなかなか見つからないことに気づいた。

GitHub のリンクカードジェネレーターはたくさんあるので、それらを参考に GitLab のリンクカードジェネレーターを作ってみた。

## 作ったもの

こういうカードを作る API。

[![bridgey-public / GitLab Card](https://gitlab-card.bridgey.workers.dev/projects/bridgey-public/gitlab-card?fullname=&link_target=_blank)](https://gitlab.com/bridgey-public/gitlab-card)

[gh-card](https://github.com/nwtgck/gh-card) を参考にして Rust で実装している。  
デプロイ先は Cloudflare Workers にした。

## 使用例

リポジトリのリンクカードを作成するには、以下のような URL にアクセスする。

```text
https://gitlab-card.bridgey.workers.dev/projects/:owner/:project
```

`:owner` と `:project` はそれぞれカードを作成したいリポジトリの値に置き換える。

- `:owner`: アカウント名、または、グループ名
- `:project`: プロジェクト名

[前回のブログ]({{<ref "/post/api-for-uploading-images-with-exif-removed/index.md">}})で紹介した `Gyazo Uploader` を例にすると、
cURL のコマンドは以下のようになる。

```bash
curl https://gitlab-card.bridgey.workers.dev/projects/bridgey-public/gyazo-uploader > test.svg
```

Markdown で GitLab プロジェクトへのリンクの付いたリンクカードを記述する例は以下のようになる。

```markdown
[![bridgey-public/gyazo-uploader](https://gitlab-card.bridgey.workers.dev/projects/bridgey-public/gyazo-uploader?fullname=&link_target=_blank)](https://gitlab.com/bridgey-public/gyazo-uploader)
```

## 注意点

- gh-card と同等の API となっていない。

  - gitlab-card は URL 末尾に拡張子を含めない。

    gh-card ではリポジトリ名のあとに `.svg` または `.png` を指定するパスパラメータとなっている。

    `worker-rs`（worker-rs が利用している `matchit`）の仕様で、パスパラメータは `:` から URL の末尾、または `/` までとなっているため、同じ構造にできなかった。

    拡張子までをリポジトリ名のフィールドに含めてしまって、`.` で分割することで同じ URL 構造のリクエストを処理するようにできるが、これは採用しなかった。  
     あまりスマートでない気がするし、同等の API にしなければならない特段の理由もないし。

  - `/repos` ではなく、`/projects` となっている。  
    GitLab の管理単位をプロジェクトと呼ぶため。

- フロントエンドの UI は未実装。

  gh-card は UI に "ユーザー名/リポジトリ名" の形式で入力することで、 HTML や Markdown 形式の文字列を取得できる。  
  gitlab-card にはまだ UI がないため、[使用例](#使用例)を参考に手作業で入力する必要がある。

  そのうち gitlab-card にも同様の UI を実装したい。
