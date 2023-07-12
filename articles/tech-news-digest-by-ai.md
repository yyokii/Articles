---
title: "AIコメンテーターにTechニュースのコメントをさせてみた"
emoji: "💃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, openai, ai]
published: true
---

## 何をつくったか

[AI+CRYPTO HACKATHON](https://ai-crypto-hack.framer.website/) が2023/06/09 ~ 06/29で開催されていたのですが、そこに提出するプロダクトとして
「Tech Digest by AI」というAIコメンテーターのコメント付きのニュースキュレーションサイトを作成しました。
（🚨 ニュースの更新は現在行なっていません）

![site top](/images/tech-news-digest-by-ai/site.png =250x)

なぜこれをつくったのか、どうやったのかを本記事ではご紹介します。

ソースはこちら 
https://github.com/yyokii/tech-digest-by-ai

## なぜつくったか

NewsPicksをよく利用しているのですが、コメンテーターにAIがいても面白いのではとふと思い作ることにしました。
ニュースが自分の興味ある領域だったり、印象のあるニュースだったりする場合に、詳細に見たいなと思いニュースサイトに行くわけですがそこにどんなコメントがついているか、というのも自分にとってはニュースサイトに訪れる要因でした。

もしAIによっていろんな人格、経歴のコメンテーターが生成され、コメントを残していたらおもしろそう！ですよね？

## 技術要素

とりあえず作ることを目標にしていたので、Next.jsのテンプレートを利用しました。

- [OpenAI API](https://platform.openai.com/)
  - 記事の選抜、記事の翻訳、記事の要約、コメンテーターの生成、コメントの生成に利用しています。
  - コンテンツ生成用の.ipynb ファイルは[こちら](https://github.com/yyokii/generator-of-ai-commented-news)
- [Next.js](https://nextjs.org/)
  - [Blog Starter Kit – Vercel](https://vercel.com/templates/next.js/blog-starter-kit)を利用し、ローカルにコンテンツを保持している静的なサイトです。
- [Tailwind CSS](https://tailwindcss.com/docs/installation)
- [DiceBear | Open Source Avatar Library](https://www.dicebear.com/)
  - コメンテーターのアイコンの生成に利用しています。

[コンテンツ生成用の.ipynb ファイル]((https://github.com/yyokii/generator-of-ai-commented-news))にて

+ RSSからのデータ取得
+ 魅力的なタイトルの抽出
+ 本文取得
+ 日本語への翻訳
+ 要約の生成
+ コメンテーターの生成
+ コメントの生成
+ .mdファイルの生成

を行い、生成された.mdを配置することで記事の更新を行なっています。

「お笑い芸人」のコメントは色が出ていて面白いのですが、他の属性のものは一般的なコメントになっているのが改善点だなあと感じました。

![comedian comment](/images/tech-news-digest-by-ai/comedian-comment.png)

## おわりに

ハッカソンの結果としては、何も賞を獲得することはできませんでした。
特に凝ったアイデアではないですし、サイト自体も一般的なものなのでそうだろうなという感じです。

しかし、AI/LLM関連で何かつくりたいなーと思っていてそれを実現できたので満足です。

この記事が誰かのためになれば幸いです。
