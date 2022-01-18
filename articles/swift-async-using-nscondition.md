---
title: "[Swift] NSConditionを用いて非同期処理を同期処理(await)に変換する"
emoji: "⏱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ios, swift]
published: false
---

[apple/swift-tools-support-core](https://github.com/apple/swift-tools-support-core) というリポジトリがあります。
そのコードを見ている際に`NSCondition`を用いてコールバックを同期実行に変換する`func`があり参考になったのでそのご紹介です。

以下の記事のようにSwiftの言語機能にあるasync/awaitを利用して変換することもできます。
[How to use continuations to convert completion handlers into async functions - a free Swift Concurrency by Example tutorial](https://www.hackingwithswift.com/quick-start/concurrency/how-to-use-continuations-to-convert-completion-handlers-into-async-functions)

## 非同期処理を同期処理に変換

以下のように`tsc_await`の引数にコールバックを持つ`func`を渡すことで同期処理に変換していました。

```swift
let manifest = try tsc_await { workspace.loadRootManifest(at: packagePath, observabilityScope: observability.topScope, completion: $0) }
```

## NSCondition

変換処理をつくるにあたって`NSCondition`の薄いラッパーが作成されています。

```swift
/// Simple wrapper around NSCondition.
/// - SeeAlso: NSCondition
public struct Condition {
    private let _condition = NSCondition()

    /// Create a new condition.
    public init() {}

    /// Wait for the condition to become available.
    public func wait() {
        _condition.wait()
    }

    /// Blocks the current thread until the condition is signaled or the specified time limit is reached.
    ///
    /// - Returns: true if the condition was signaled; otherwise, false if the time limit was reached.
    public func wait(until limit: Date) -> Bool {
        return _condition.wait(until: limit)
    }

    /// Signal the availability of the condition (awake one thread waiting on
    /// the condition).
    public func signal() {
        _condition.signal()
    }

    /// Broadcast the availability of the condition (awake all threads waiting
    /// on the condition).
    public func broadcast() {
        _condition.broadcast()
    }

    /// A helper method to execute the given body while condition is locked.
    /// - Note: Will ensure condition unlocks even if `body` throws.
    public func whileLocked<T>(_ body: () throws -> T) rethrows -> T {
        _condition.lock()
        defer { _condition.unlock() }
        return try body()
    }
}
```

## 変換処理

同期処理へ変換する`func`は次のようになっています。

```swift
/// Converts an asynchronous method having callback using Result enum to synchronous.
///
/// - Parameter body: The async method must be called inside this body and closure provided in the parameter
///                   should be passed to the async method's completion handler.
/// - Returns: The value wrapped by the async method's result.
/// - Throws: The error wrapped by the async method's result
public func tsc_await<T, ErrorType>(_ body: (@escaping (Result<T, ErrorType>) -> Void) -> Void) throws -> T {
    return try tsc_await(body).get()
}

public func tsc_await<T>(_ body: (@escaping (T) -> Void) -> Void) -> T {
    let condition = Condition()
    var result: T? = nil
    // 処理の実行
    body { theResult in
        condition.whileLocked { // ロックした状態で処理を実行（抜ける時にロックを解除）
            result = theResult  // 結果を取り出す
            condition.signal() // スレッドへ通知
        }
    }
    condition.whileLocked {
        while result == nil {
            condition.wait() // 結果を未取得の場合はスレッドをブロック
        }
    }
    return result!
}
```

処理のフローは次のようになっています。

`body {~}`で処理を実行し、結果を取得できるまで`wait`します。
そして、値が取得できたら`signal`で通知を行い待機しているスレッドを起動状態に更新します。



