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

Swiftでのこの問題への対応はロック処理の記載、Serial queueの作成が考えられますがActorを用いることでよりシンプルにそれを解決することができます。つまりあるオブジェクトへのアクセスが一度に単一のタスクに制限する必要がある場合、Actorは有効です。Actorを宣言することで同時に1つの処理のみがデータにアクセスすることを保証されますが、これはActor隔離（Actor isolated）と呼ばれます。

Actorは上記の特徴に加え、

* 参照型
* 継承はできない

という性質も持ちます。

また、Actorは再入可能性（reentrancy）を持ちます。

>  When an actor-isolated function suspends, reentrancy allows other work to execute on the actor before the original actor-isolated function resumes, which we refer to as *interleaving*. 
>
> （翻訳）
>
> アクターから分離された関数が一時停止した場合、リエントランシーにより、元のアクターから分離された関数が再開する前に、アクター上で他の作業が実行されます（これを*インターリーブと*呼びます）。

[引用: swift-evolution/0306-actors.md at main · apple/swift-evolution](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy)

これにより、

* デッドロックになることをケア
* アクター上の処理を不必要にブロックしないことで全体的なパフォーマンスを向上

できます。

一方で、再入可能性によりawaitの前後で状態が変わる可能性があります。（= 競合状態（race condition））
例えば以下のコードでは`count`の値は最初は0ですが、await後は1になります。

```swift
actor Demo {
    var count = 0

    func doWork() async {
        print("📝 before sleep count: \(count)")

        // awaitにより他のタスクがここまで到達する可能性がある
        try! await Task.sleep(nanoseconds: 5_000_000_000)

        // countの値がawaitの前後で一致しない可能性がある
        print("📝 after sleep count: \(count)")
    }

    func increment() {
        count += 1
    }
}

let demo = Demo()

Task {
    await demo.doWork()
}

Task {
    await demo.increment()
}
```

```
（出力）
📝 before sleep count: 0
📝 after sleep count: 1
```

この問題への対応としては以下の対応が可能です。

* 状態変更を同期的に（awaitを挟まずに）実行する
* await後に再度状態を確認する

### isolated/nonisolated キーワード

Actorの中で定義されたfuncはデフォルトでisolated（Actor隔離）となります。

isolatedキーワードは引数に付与することも可能です

```swift
actor User {
    var name = "Swift"
    var height: Int?
    var weight: Int?
}

// isolatedをつけることで関数全体がsuspendポイントになる
func doWork(user: isolated User) {
    print("name: \(user.name)")
    print("height: \(user.height ?? 0)")
    print("weight: \(user.weight ?? 0)")
    await Task.sleep(2)
}

// 個別にawaitする例
func doWork(user: User) async {
    print("name: \(await user.name)")  // 個々のアクセスでawaitが必要
    print("height: \(await user.height ?? 0)")
    print("weight: \(await user.weight ?? 0)")
}


Task {
    let user = User()
    await doWork(user: user)
}

```

nonisolatedを用いることでActor隔離を行わないようにできます。
以下の特徴があります。

* Actor隔離をしないことで、awaitを利用せずともアクセスできる
* let 宣言のプロパティはnonisolatedを明示しなくてもActorの外からアクセス可能
* Actorを特定のプロトコル（HashableやCodable）に準拠させる際に用いられることもある
  * 同期的なアクセスを要求するものに対して、nonisolatedをつけることでコンパイルが可能となる

### Actor hopping

Actor hoppingとはスレッドがあるActorの処理を中断し別のActorの処理を開始することです。

MainActorとそれ以外のAcotrの間でこれが生じた場合、MainActorは協調スレッドプールにないのでコンテキストスイッチが発生します。
そしてこれが頻繁に生じるとパフォーマンス低下に繋がります。
例えば、MainActorから他のActorの処理を複数回呼び出している場合は、インターフェイスを変更するなどして呼び出し回数を減らすことは有効です。

[参考: What is actor hopping and how can it cause problems? - a free Swift Concurrency by Example tutorial](https://www.hackingwithswift.com/quick-start/concurrency/what-is-actor-hopping-and-how-can-it-cause-problems)

## MainActor

MainActorはプロパティへのアクセスやメソッドの実行がメインキューにて行われるActorです。
UIViewControllerやUIViewなどにも付与されています。

また、`@StateObject`や`@ObservedObject`を用いた際は暗黙的にその`struct`や`class`は`@MainActor`が付与されます。これはそれらのプロパティラッパーの`wrappedValue`に`@MainActor`が付与されているためです。

> - A struct or class containing a wrapped instance property with a global actor-qualified `wrappedValue` infers actor isolation from that property wrapper:
>
>   ```swift
>   @propertyWrapper
>   struct UIUpdating<Wrapped> {
>     @MainActor var wrappedValue: Wrapped
>   }
>     
>   struct CounterView { // infers @MainActor from use of @UIUpdating
>     @UIUpdating var intValue: Int = 0
>   }
>   ```

[引用: swift-evolution/0316-global-actors.md at main · apple/swift-evolution](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md)

関数がすでにMainActorで実行されている場合、`await MainActor.run()`を使用すると、次の実行ループを待たずにすぐにコードが実行されますが、`Task { @MainActor in ~ `使用*すると、*次の実行ループを待つことになります。

[参考: How to use @MainActor to run code on the main queue - a free Swift Concurrency by Example tutorial](https://www.hackingwithswift.com/quick-start/concurrency/how-to-use-mainactor-to-run-code-on-the-main-queue)

### MainActorブロッキング

？？

### inference work

明示的にMainActorや`nonisolated`を宣言していなくても推論が働きます。

* MainActorなもののサブクラスは自動でMainActorとなる
* MainAcor宣言であるメソッドのオーバーライドは自動でMainActorなメソッドとなる
* property wrapperにおいてwrapped valueにMainActorを付与しているもの（@ObservedObjectや@StateObject）を利用するclassやstructは自動でMainActorとなる
* ProtocolのメソッドがMainActorである場合、その実現メソッドは自動でMainActorとなる
  * が、Protocol準拠とは別で実装した場合（例えばextensionを別途作成した場合）はMainActorにはならない
* Protocol自体がMainActorである場合、それに準拠したものは自動でMainActorとなる（メソッドも含む）
  * が、別ファイルで準拠した場合はメソッドのみが自動でMainActorとなる

[参考: swift-evolution/0316-global-actors.md at main · apple/swift-evolution](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md#global-actor-inference)

## Sendable

データ競合が生じない（スレッドセーフである）ことをコンパイル時に保証するマーカープロトコル。
同期処理が必要ない、ということを明示できます。

> Values of types that conform to the `Sendable` protocol are safe to share across concurrently-executing code.

[swift-evolution/0306-actors.md at main · apple/swift-evolution](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#cross-actor-references-and-sendable-types)

IntやStringなどのメタタイプ、Actor、Struct、enumなどはSendableな型です。

SendableがあることでActorへ値を渡すときなどに、データ競合が起きないことをコンパイル時に保証できます。

## Async/Await

非同期処理を作成/使用する際のキーワードです。
await（= suspend）することでスレッドをブロックせず他の処理実行を可能にします（一方で同期処理の場合はブロッキングになる）。awaitであるからといって必ずsuspendするかは分からず、その可能性があるというだけです。

また、suspend状態から戻った時に必ずしも以前と同じスレッドで処理が実行されるとは限りません。

読み取り専用のコンピューティッドプロパティにおいても、async/awaitを利用可能
https://www.hackingwithswift.com/quick-start/concurrency/how-to-create-and-use-async-properties

## Unstructured Concurrency

async/awaitだけでは逐次実行されるだけで並行処理は実現できません。
Swiftにでは、Unstructured ConcurrencyやStructured Concurrencyを用いることで並行処理を実現できます。
まずはUnstructured Concurrencyを見ていきます。

### Task.init vs Task.detached

タスクは作成後すぐに時実行されます。
タスクの生成には`.init`か`.detached`を利用できますが、Priority、Task local call、Actor contextの扱いに違いがあります。

|               | Priority            | Task local value     | Actor context        |
| ------------- | ------------------- | -------------------- | -------------------- |
| Task          | 引き継ぐ            | 引き継ぐ             | 引き継ぐ             |
| Task.detached | nil（未指定の場合） | なし（引き継がない） | なし（引き継がない） |

（タスクのPriorityについてはhigh, medium, .lowがありますが、nilを設定するとmediumとなるようです。
[swift - Default priority of Task.detached - Stack Overflow](https://stackoverflow.com/a/73513116/9015472)）

> First off, `Task.detached` most of the time should not be used at all, because it does *not* propagate task priority, task-local values or the execution context of the caller. Not only that but a detached task is inherently not *structured* and thus may out-live its defining scope.
>
> （翻訳）
>
> まず、Task.detachedは、タスクの優先順位やタスクローカルな値、呼び出し元の実行コンテキストを伝搬しないので、ほとんどの場合、使用すべきではありません。それだけでなく、デタッチドタスクは本質的に構造化されていないため、定義されたスコープを超える可能性があります。

[引用: swift-evolution/0317-async-let.md at main · apple/swift-evolution](https://github.com/apple/swift-evolution/blob/main/proposals/0317-async-let.md#comparing-with-task-and-not-proposed-futures)

例えば、Task.detachedを用いた際にはActor contextを引き継がないのでActor内のfuncを利用する場合は`await`が必要になります。

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

`.detached`なタスクは親タスクから完全に独立したタスクとなり、構造化されたタスクではないので協調的にキャンセルが起こることもありません。
[Explore Structured Concurrency in Swiftの](https://developer.apple.com/videos/play/wwdc2021/10134/)において利用例として、ネットワークから画像をダウンロードし、キャッシュする際に利用するケースが紹介されています。画像取得後にダウンロードタスクのキャンセルの影響を受けずに、キャッシュ処理を行いたいのでこのような利用が可能でしょう。

### Taskから結果を取得

Taskの結果を取得するこが可能です。
`.result`を用いることで` Result<Success, Failure>`を取得でき、また`.value`で`Success`時の値のみを取得可能です。

```swift
func demo() async {
    let doTask = Task { () -> String in
        return "hi"
    }
    
    // Result型を取得。.valueを用いれば成功時の値のみを取得可能。
    let result = await doTask.result

    do {
        let data = try result.get()
    } catch {
        print(error)
    }
}
```

### キャンセル

Taskは`cancel()` でキャンセルができ、その検知は`Task.isCancelled`で可能です。
`Task.checkCancellation()`を呼ぶことでキャンセル時に`CancellationError`を投げることもできます。

[参考: How to cancel a Task - a free Swift Concurrency by Example tutorial](https://www.hackingwithswift.com/quick-start/concurrency/how-to-cancel-a-task)

`Task.sleep()`を呼び出すと、自動的にキャンセルがチェックされます。つまりタスクをキャンセルすると、`CancellationErrorが`投げられそれをキャッチすることができます。

### yeild

> A task can voluntarily suspend itself in the middle of a long-running operation that doesn’t contain any suspension points, to let other tasks run for a while before execution returns to this task.
>
> If this task is the highest-priority task in the system, the executor immediately resumes execution of the same task. As such, this method isn’t necessarily a way to avoid resource starvation.
>
> （翻訳）
>
> サスペンドポイントを含まない長時間の処理の途中で自発的にサスペンドし、このタスクに実行が戻るまでの間、他のタスクを実行させることができます。
>
> このタスクがシステム内で最も優先度の高いタスクである場合、実行者は直ちに同じタスクの実行を再開する。そのため、この方法は必ずしもリソース枯渇を回避する方法とは言えません。

`Task.yield()`は、非同期タスクから実行を一時停止して、他のタスクが実行されるようにするために使用されます。実行中のタスクが次のタスクに制御を渡す可能性があることを示しますが、現在のタスクの優先度が高ければそれを継続して行います。

 ## Structured Concurrency

> タスクは階層を構築します。タスクグループ内の各タスクは同じ親タスク(*parent task*)を持ち、各タスクは子タスクを持つことができます。タスクとタスクグループの間の明示的な関係のために、このアプローチは構造化された並行性(*structured concurrency*)と呼ばれます。

[同時並行処理(Concurrency) · The Swift Programming Language日本語版](https://www.swiftlangjp.com/language-guide/concurrency.html)

タスクグループは`withTaskGroup`や`withThrowingTaskGroup`で作成できます。
タスクは完了順で返却されあmす。

例えば次のようにTask groupを作成できます。

```swift
func doWork() async {
    let string = await withTaskGroup(of: String.self) { group -> String in
        group.addTask { "one" }
        group.addTask { "two" }
        group.addTask { "three" }
        group.addTask { "four" }

        // 全てのタスクを待つ
        var results: [String] = []
        for await value in group {
            results.append(value)
        }
        return results.joined(separator: ",")
    }

    print(string)
}

Task {
    await doWork()
}
```

タスクを待ち受ける際は以下のようにできます。

* （上述のように）for loopを用いて子タスクの完了を待つ
* `next()`を用いる
  * 次の子タスクを待つ（もちろんtaskの完了順です）
* ``waitForAll(`)`を用いる
  * 全てが完了するまで待ち、且つ戻り値は破棄される

### キャンセル

Structured Concurrencyでは以下の場合にタスクのキャンセルが発生します。

* 親タスクがキャンセルされると自動でキャンセルされる

* `group.cancelAll()`を呼び出した場合

* 子タスクがエラーを投げ、且つそれがキャッチされない場合

  * ```swift
    group.addTask {
        try await Task.sleep(nanoseconds: 1_000_000_000)
        throw DemoError.other
    }
    ```

`cancellAll()`を呼び出しても、例えば下記のようにキャンセルの処理をしなければタスクの出力は変わりません。

```swift
group.addTask {
    try Task.checkCancellation()
    return "Testing"
}
```

`group.cancelAll()`は未完了のタスクをキャンセルするので、すでに実行済みの処理は取り消しません。

上述のようにキャンセルできますが、グループに対して`addTask()`を呼び出すと、すでにグループをキャンセルしていたとしても、無条件で新しいタスクがグループに追加されます。キャンセルされたグループにタスクを追加したくない場合は、`addTaskUnlessCancelled()`を利用します。

### async let

複数のタスクを並列で実行する方法として`async let`もあります。

|                | 実行タスク数 | 結果の取得     | 直接のキャンセル | タスクの戻り値                                               |
| -------------- | ------------ | -------------- | ---------------- | ------------------------------------------------------------ |
| `async let`    | 固定数       | 呼び出し順保持 | ×                | タスクごとに設定可能                                         |
| タスクグループ | 可変個       | 完了順         | `cancelAll()`    | タスクの戻り値を別の型にするにはenumを利用するなどの対応が必要 |

> タスクの戻り値を別の型にするにはenumを利用するなどの対応が必要

これについては、以下のように結果のパターンをenumで表しその中で戻り値を表すようにすれば可能です。

> ```swift
> // A single enum we'll be using for our tasks, each containing a different associated value.
> enum FetchResult {
>  case username(String)
>  case favorites(Set<Int>)
>  case messages([Message])
> }
> ```

[How to handle different result types in a task group - a free Swift Concurrency by Example tutorial](https://www.hackingwithswift.com/quick-start/concurrency/how-to-handle-different-result-types-in-a-task-group)

## SwiftUIとSwift Concurrency

SwiftUIの`.task()`modifierを使用してタスクを実行できます。
この場合ビューが消えるときにそのタスクは自動的にキャンセルされます。

また、`task(id:priority:_:)`というidを引数にとるものを使うことでその値が更新された場合にタスクを実行することができます。

> id
> The value to observe for changes. The value must conform to the [`Equatable`](https://developer.apple.com/documentation/Swift/Equatable) protocol.

https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:)





----

* 例えば、`data(from:delegate:)`などは自動的にキャンセル処理を行わないが、なぜそうしているのだろうか？ Task.checkCancellation() を流すという考えもありそうだが？
  * キャンセル処理を自動化すると、APIの動作が予測しづらくなる可能性があるため、利用者にキャンセル処理を任せる方が良い。
  * キャンセル処理はアプリケーションのコンテキストに合わせて様々な方法で行われるため、API側でキャンセル処理を自動化しない方がベター。
  * キャンセル処理を自動化する場合、キャンセルリクエストが受け取られた時点で、その処理に関するすべてのリソースを解放する必要があるため、処理が完了するまで待たなければならず、即座にキャンセル処理を行うことができない場合がある。

* `@State`プロパティのラッパーは、どのスレッドでもその値を変更できるように特別に記述されています。

* `withCheckedContinuation`, `withCheckedThrowingContinuation`による変換
  * `CheckedContinuation<T, E> where E : Error`を利用してdelegateパターンにも対応可能
  * resume()メソッドを呼ぶ



https://swiftsenpai.com/swift/actor-reentrancy-problem/



（📝  wwdcの動画見た方がいいかも、協調スレッドプール、actor hoppingについて）



https://www.wwdcnotes.com/notes/wwdc21/10134/
