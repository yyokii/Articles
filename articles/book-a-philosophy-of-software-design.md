---
title: "「A Philosophy of Software Design」のまとめと感想"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [書籍, 設計, システム設計, 設計原則 ]
published: true
---

## A Philosophy of Software Design
「A Philosophy of Software Design」はソフトウェア開発の設計における問題とその対処法を紹介している本です。本の中では「Red Flag」という設計原則がいくつか示されており、それによってより良い設計をすることを目指しています。
評価が高い本であり、自分も参考になりましたので各章のまとめ（スライド）と感想を記事にしました。

尚、本記事におけるチャプター毎の要約に関しては、著者の許諾を得て公開しています。

## 各章の概要

### Chapter 1
良いデザインをすることが大事であり、そのためには複雑さを減らしてく必要があります。
本書では複雑さの詳細とその対処について書かれています。
![chapter1](/images/APOSD-001.png)

### Chapter 2
複雑さとは何か？
![chapter2](/images/APOSD-002.png)

### Chapter 3
良いデザインをするための心構え
![chapter3](/images/APOSD-003.png)

### Chapter 4
Deep Moduleによる複雑さの隠蔽
![chapter4](/images/APOSD-004.png)

### Chapter 5
情報の隠蔽を意識する
![chapter5](/images/APOSD-005.png)

### Chapter 6
汎用化により特殊性を排除して複雑さを減らす
![chapter6](/images/APOSD-006.png)

### Chapter 7
レイヤー毎に異なる抽象化をする
![chapter7](/images/APOSD-007.png)

### Chapter 8
複雑さを引き下げることを考える
![chapter8](/images/APOSD-008.png)

### Chapter 9
分割と統合の基準
![chapter9](/images/APOSD-009.png)

### Chapter 10
特殊ケースの大きな要因である例外への対処
![chapter10](/images/APOSD-010.png)

### Chapter 11
良いデザインをするめに「Design it Twice」
![chapter11](/images/APOSD-011.png)

### Chapter 12
なぜコメントを書くべきなのか
![chapter12](/images/APOSD-012.png)

### Chapter 13
どんなコメントを書くべきなのか
![chapter13](/images/APOSD-013.png)

### Chapter 14

良い命名
![chapter14](/images/APOSD-014.png)

### Chapter 15
先にコメント書く
![chapter15](/images/APOSD-015.png)

### Chapter 16
コメントの保守
![chapter16](/images/APOSD-016.png)

### Chapter 17
一貫性をとる
![chapter17](/images/APOSD-017.png)

### Chapter 18
コードを明白なものにして曖昧さを省く
![chapter18](/images/APOSD-018.png)

### Chapter 19
最近のソフトウェアの動向は複雑さをどう解消するか
![chapter19](/images/APOSD-019.png)

### Chapter 20
クリーンでシンプルなコードはパフォーマンスにも良い
![chapter20](/images/APOSD-020.png)

### Chapter 21
重要なものの見つけ方
![chapter21](/images/APOSD-021.png)

### Chapter 22
![chapter22](/images/APOSD-022.png)

## 本の感想

「Red Flag」（本書で示されている設計原則）が役に立つのはもちろんのこと、失敗例が随所で紹介されており、それによって過去の自分が犯した同じ失敗が想起されて、本書の内容がより心に刻まれたように思います。
また、自分の経験からも思うのですが、汎用的でシンプルな設計ができるように、設計をリードする人はプロダクトのビジネス観点の話に積極的に参加する必要性を感じました。ビジネスチームとコミュニケーションを取ることでソフトウェアが解決しようとしているものを正確に認識でき、またこれからの展望もわかります。それは全体の見通しを持つことにつながり、より良い設計ができます。自分の経験でもビジネスチームとのコミュニケーションが密な時の方が良い設計ができていたように思います。
本書を通じて使いやすいソフトウェアを作るということはプロダクトについて考えるだけでなく、開発しやすいソフトウェアにするということも考えることが重要だと改めて感じました。その意味で、エンジニアが意識すべき「ユーザー」はそのシステムの利用者だけではなく、自分の書いたコードを読む将来の自分や他の開発者も含まれるのだと思いました。

チーム全体でコミュニケーションをうまく取り、本書の教えを実践していけるよう頑張っていこうと思えた素敵な本でした。

---

2023/01/29追記

[CTO から見た，なぜスタートアップ 初期のソフトウェア設計は壊れがちなのか - Speaker Deck](https://speakerdeck.com/memory1994/why-the-application-design-is-breaking-sometimes-at-a-startup-company)

こちらの資料で語られているスタートアップでのプロダクト開発、エンジニア採用の話は本書で話されているStrategic vs Tacticalの話と同じである。会社のフェーズ、他社状況を鑑みて開発戦略、採用戦略、社内の評価制度をうまくコントロールしていく必要がある。スタートアップの初期においては特にリードするエンジニアはビジネスの状況を意識し、それをソフトウェアの設計に落とし込む必要がある。