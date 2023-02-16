---
title: "[Swift] Swift Conccurenyをざっくりと"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

ざっと雑に書いて、改めて記事等を見て付け加える感じが良さそう

---

Swift Concurrencyで出てくるキーワードについてのまとめです。

## 全体感

## Actor

プログラミングにおいてデータ競合（data race）という問題があります。
これは特定の変数について、複数スレッド間から同時に読み書き（少なくとも1つは書き込み処理）が行われることで生じます。

Swiftでのこの問題への対応はロック処理の記載、Serial queueの作成が考えられますがActorを用いることでよりシンプルにそれを解決することができます。

Actorを宣言することで、同時に1つの処理のみがデータにアクセスすることを保証します。（= Actor隔離（Actor isolated）と呼ぶ

再入可能性

### isolated/nonisolated キーワード

* Actorの中で定義されたfuncはデフォルトでisolatedです
* nonisolatedを用いることでActor隔離を行わないようにできる
  * Actor隔離をしないことで、awaitを利用せずともアクセスできる
  * let 宣言のプロパティはnonisolatedを明示しなくてもActorの外からアクセス可能
  * Actorを特定のプロトコル（HashableやCodable）に準拠させる際に用いられることもある
    * 同期的なアクセスを要求するものに対して、nonisolatedをつけることでコンパイルが可能となる



### Actor hopping

Actor hoppingとはスレッドがあるActorの処理を中断し別のActorの処理を開始することです。

MainActorとそれ以外のAcotrの間でこれが生じた場合、MainActorは協調スレッドプールにないのでコンテキストスイッチが発生します。（📝  wwdcの動画見た方がいいかも、協調スレッドプール、actor hoppingについて）
そしてこれが頻繁に生じるとパフォーマンス低下に繋がります。
例えば、MainActorから他のActorの処理を複数回呼び出している場合は、インターフェイスを変更するなどして呼び出し回数を減らすことは有効となるでしょう。

https://www.hackingwithswift.com/quick-start/concurrency/what-is-actor-hopping-and-how-can-it-cause-problems
https://zenn.dev/yimajo/articles/e993344237f443

（📝 やはり behind the scenesのやつが良さそう）

## MainActor

プロパティへのアクセスやメソッドの実行がメインキューにて行われるActor。

UIViewControllerやUIViewなどにも付与されている。

### MainActorブロッキング

### inference work

* MainActorなもののサブクラスは自動でMainActorとなる
* MainAcor宣言であるメソッドのオーバーライドは自動でMainActorなメソッドとなる
* property wrapperにおいてwrapped valueにMainActorを付与しているもの（@ObservedObjectや@StateObject）を利用するclassやstructは自動でMainActorとなる
* ProtocolのメソッドがMainActorである場合、その実現メソッドは自動でMainActorとなる
  * が、Protocol準拠とは別で（例えばextensionを別途作成した場合）はMainActorにはならない
* Protocol自体がMainActorである場合、それに準拠したものは自動でMainActorとなる（メソッドも含む）。
  * が、別ファイルで準拠した場合はメソッドのみが自動でMainActorとなる
  * これによりあるProtocolがMainActorになったとしても既に準拠しているものが強制的にMainActorにはならない。

### 実行ループ

## Sendable

データ競合が生じないことをコンパイル時に保証するマーカープロトコル。

> Values of types that conform to the `Sendable` protocol are safe to share across concurrently-executing code.

https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#cross-actor-references-and-sendable-types

* メタタイプ
* tuple
* Actor
* struct
* enum

### 暗黙的な準拠ルール

## Async/Await

非同期処理を作成/使用する際のキーワード

await（= suspend）することどスレッドをブロックせず他の処理実行を可能にする
（awaitであるからといって必ずsuspendするかは分からず、その可能性があるというだけ）

computed propertyもasyncにすることが可能（getのみ）

suspend状態から戻った時に必ずしも以前と同じスレッドで処理が実行されるとは限らない

## Unstructured Concurrency

async/awaitだけでは逐次実行されるだけで並行処理は実現できない。

📝 https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#motivation この例も同じこと言ってるはず↑

並行処理をするために、TaskとTask groupを用いる。

> A unit of asynchronous work.

https://developer.apple.com/documentation/swift/task

即時実行される。

### Task.init vs Task.detached

> First off, `Task.detached` most of the time should not be used at all, because it does *not* propagate task priority, task-local values or the execution context of the caller. Not only that but a detached task is inherently not *structured* and thus may out-live its defining scope.

|               | Priority | Task local value     | Actor context        |
| ------------- | -------- | -------------------- | -------------------- |
| Task          | 引き継ぐ | 引き継ぐ             | 引き継ぐ             |
| Task.detached | 0        | なし（引き継がない） | なし（引き継がない） |

Actor contextの例

```swift
actor Demo {
    func doWork() {
        Task.detached {
            if await self.validate() {　// detachedにしているので同じActor内の処理についてawaitが必要
                // do something
            }
        }
    }

    func validate() -> Bool {
        return true
    }
}
```

* 優先度
  * high: .userInitiatedと同義
  * medium: nilを設定するとこれになる
    * https://stackoverflow.com/a/73513116/9015472
  * low: .utilityと同義

### Taskから結果を取得

```swift
func demo() async {
    let doTask = Task { () -> String in
        return "hi"
    }

    let result = await doTask.result

    do {
        let data = try result.get()
    } catch {
        print(error)
    }
}
```

https://developer.apple.com/documentation/swift/task/result

 ## Structured Concurrency





https://www.wwdcnotes.com/notes/wwdc21/10134/
