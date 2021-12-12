---
title: "[Server-Side Swift] Vaporで定期実行してiOS向けの情報をまとめてみた"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [swift, ios]
published: false
---

（本記事は[ZOZO Advent Calendar 2021](https://qiita.com/advent-calendar/2021/zozo)のカレンダー5の15日目です。）

Swiftでサーバーサイドも体験してみたくなったのでVaporで簡単なものをつくってみました。
そのご紹介です。

## つくったもの

[yyokii/AppleDeveloperNews](https://github.com/yyokii/AppleDeveloperNews)

* Appleの開発者、とりわけiOSアプリ開発者向けの情報をまとめています
* 毎週日曜に自動更新

## 目的

* Vaporを触ってみたかった
* Async/Awaitを使いたかった
* 気になる情報は定期的に追いたいと思いつつも、気づいたらベースでみるようになっていたので、自分で作ればみるかなと思いつくってみた

## 技術的ポイント

### Async/Await

Swift5.5から使えるasync/awaitキーワードを用いた非同期プログラミングが可能です。

Vaporには非同期プログラミングをサポートするために[SwiftNIO](https://github.com/apple/swift-nio)を取り入れています。
しかし、Swift 5.5からは Concurrency の言語機能が追加されたためasync/awaitキーワードを用いて書くことができるようになりました。

例えば以下のようにSwiftNIOベースの`EventLoopFuture`に基づいた記述があるとすると

```swift
return someMethodCallThatReturnsAFuture().flatMap { futureResult in
    // use futureResult
}
```

次のように書き換えることができます。（[Vapor: Basics → Async migrating-to-asyncawait](https://docs.vapor.codes/4.0/async/#migrating-to-asyncawait) より）

```swift
let futureResult = try await someMethodThatReturnsAFuture().get()
```

どちらの書き方を採用するかについてですが、

* `EventLoopFuture` : Event Loopを制御したい場合 or 高いパフォーマンスを期待する場合
* `async`/`await` : 多くの場合これで事足りる。パフォーマンスは劣るが、その機能や可読性がもたらすメリットの方が大きい

ということのようです。

> For applications that need explicit control over event loops, or very high performance applications, you should continue to use `EventLoopFuture`s until custom executors are implemented. For everyone else, you should use `async`/`await` as the benefits or readability and maintainability far outweigh any small performance penalty.

[Vapor: Basics → Async](https://docs.vapor.codes/4.0/async/)

ということでConcurrencyのキャッチアップとしてVaporで試してみるのも良さそうです。
また、iOSアプリ開発などでConcurrencyを使っている人はその知識をそのまま利用できて良いですね。

自分もAPIクライアントを作成してそこで`async/await`を利用しようとしていましたが、Vaporには[標準のクライアント](https://docs.vapor.codes/4.0/client/)があることを後で知りました。
ですのでコードはそちらを利用した記述に修正予定です。

また、[swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation/blob/eec4b26deee34edb7664ddd9c1222492a399d122/Sources/FoundationNetworking/URLSession/URLSession.swift)（Darwin以外のプラットフォームではFoundationを使用できるようにするためのSwiftライブラリ）とローカルで開発しているときに利用できるAPIとでは差異がある（swift-corelibs-foundationが少し遅い）のでDockerなどでデプロイ先の環境を実現しそこでビルドやテスト実行しておくことが大事と感じました。

### スケジューリング

スケジューリングするためにVaporの[Queues](https://docs.vapor.codes/4.0/queues/#scheduling-jobs)を利用しました。
現状公式にサポートしているドライバーは[QueuesRedisDriver](https://github.com/vapor/queues-redis-driver)です。（従ってRedisを利用しています。）

`ScheduledJob` もしくは `AsyncScheduledJob `に準拠したものをつくり、それを登録することでジョブのスケジューリングができます。

名前からわかるように`AsyncScheduledJob `は `Concurrenc`機能に対応しています。

コンテンツの生成とgithubの更新ジョブは次のように記述しています。

```swift
struct GenerateAppleDevNewsJob: AsyncScheduledJob {
    init() {}
    
    func run(context: QueueContext) async throws {
        let content = try await generateMdContent()
        try await updateGitHub(content: content.content)
    }
  .
  .
  .
```

configure（アプリの設定処理を書くところ）内

```swift
    let generateAppleDevNews = GenerateAppleDevNewsJob()
    app.queues.schedule(generateAppleDevNews)
        .weekly()
        .on(.sunday)
        .at(.noon)
    
    try app.queues.startScheduledJobs()
```

### データの取得

RSSとしてデータ提供がされていないものがあったのでいくつかはスクレイピングをしています。

#### RSSによる取得

RSSのレスポンスをパースするのは面倒だったので、[RSS to JSON Converter online](https://rss2json.com/#rss_url=https%3A%2F%2Ftechcrunch.com%2Ffeed%2F)を利用しレスポンスをJSON化して利用しました。

#### スクレイピング

HTMLパーサーとして[SwiftSoup](https://github.com/scinfu/SwiftSoup)を利用しました。
Swift製のHTMLパーサーでありSwift Package Managerへ対応し、スターは3Kほどあります。 
今回は特定ページから特定コンテンツを取得するというシンプルな利用でしたが、直感的に使えて便利でした。

## おわりに

触ってみるとやはり楽しいので今後も何かつくっていけたらなと思いました。
Server-Side Swiftを採用したプロダクトは他の言語に比べるとかなり稀有なのは少し残念ですね。

## 参考

* [Server-Side Swift with Vapor | raywenderlich.com](https://www.raywenderlich.com/books/server-side-swift-with-vapor)
  * Vaporのキャッチアップとして読みました。丁寧な説明でありまた応用的なとこまでカバーされており良かったです。
* [Vapor Docs](https://docs.vapor.codes/4.0/)
  * 公式ドキュメントです。
