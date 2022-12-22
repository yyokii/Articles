---
title: "[Mac App] TCAを使ってシンプルなMacアプリを作成してみた"
emoji: "🍅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ios, swift, tca, mac]
published: true
---

:::message 
これは [ZOZO Advent Calendar 2022](https://qiita.com/advent-calendar/2022/zozo) カレンダー Vol.1 の 23 日目の記事です。
:::

## 何をつくったか

![pomodoro app](/images/pomodoro-mac-app.png)

TCA, SwiftUI, Firebaseを利用してシンプルなポモドーロタイマーアプリを作成しました。
機能としては以下があります。

* タイマー機能
* ポモドーロの設定変更
* タイマーリセット
* 匿名サインイン
* サーバーにデータ保存

これ以外にも

* 過去データ閲覧
* 他のカレンダーとの同期

機能を追加しようと思っていましたが、作るに至りませんでした。

全体のコードはこちらにあります。
https://github.com/yyokii/PomodoroApp

## アプリの構成

クライアント：SwiftUIとTCA
バックエンド：Firebase（認証処理、Firestore）

## 詳細

### AppCore

アプリ全体の ステートを管理するために、AppCoreのReducerで各機能のアクションをハンドリングするようにしました。
例えばユーザーが変更したりある画面で変更を起こしたりした際に、連動して他の画面へも反映が発生する場合、AppCoreからstateを更新することでそれを実現できます。
現状はシンプルなアプリ、機能のためほぼ何もしていません。

```swift
public struct AppState: Equatable {
    var account: AccountState = AccountState()
    var myData: MyDataState = MyDataState()
    var pomodoroTimer: PomodoroTimerState = PomodoroTimerState()
    public var settings: SettingsState = SettingsState()

    public init() {}
}

public enum AppAction: Equatable {
    case account(AccountAction)
    case myData(MyDataAction)
    case pomodoroTimer(PomodoroTimerAction)
    case settings(SettingsAction)
    ...
}
```

### Binding

SwiftUIで書いているとViewとデータをBindingすることができますが、TCAでも可能です。
TCAでもBindingは利用可能ですが少し特殊な書き方になります。

Stateにて、ViewにBindingしたものに`@BindableState`をつけます。そうすることで、例えばViewでテキストフィールドやピッカーでこれらの値を直接更新できることを宣言します。

```swift
public struct PomodoroTimerSettingsState: Equatable {
    @BindableState var intervalTime: Int = 0
    @BindableState var shortBreakTime: Int = 0
    @BindableState var longBreakTime: Int = 0
    @BindableState var intervalCountBeforeLongBreak: Int = 0

    public init() {}
}
```

値が更新された場合に処理を実行できるようにActionにbindingケースを追加します。

```swift
public enum PomodoroTimerSettingsAction: Equatable, BindableAction {
    case binding(BindingAction<PomodoroTimerSettingsState>)
    case onAppear
    case onDisappear
}
```

Reducerでは`.binding()`を付与してBindingを処理できるようにします

```swift
public let pomodoroTimerSettingsReducer: Reducer<PomodoroTimerSettingsState, PomodoroTimerSettingsAction, PomodoroTimerSettingsEnvironment> = .combine(
    .init { state, action, environment in
        switch action {
        case .binding:
            return environment.userDefaults.setPomodoroTimerSettings(.init(
                intervalSeconds: state.intervalTime * 60,
                shortBreakIntervalSeconds: state.shortBreakTime * 60,
                longBreakIntervalSeconds: state.longBreakTime * 60,
                intervalCountBeforeLongBreak: state.intervalCountBeforeLongBreak)
            )
                .fireAndForget()
        ...
        }
    }
).binding()
```

以上のようにすることでViewでは以下のようにしてBindingを実現できます。

```swift
Picker("", selection: viewStore.binding(\.$intervalTime)) {
    ...
}
```

詳細は以下の記事を参照してみて下さい。

* [The Composable Architecture ❤️ SwiftUI Bindings](https://www.pointfree.co/blog/posts/63-the-composable-architecture-%EF%B8%8F-swiftui-bindings)
* [Deprecate dynamic member lookup on view stores in favor of ViewStore.binding by stephencelis · Pull Request #810 · pointfreeco/swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture/pull/810)

### Firebase Client

Firebaseの簡易なClientを作成しEffectを受け取れるようにしました。
オフライン状態が発生してもオンライン復帰時に同期してくれるので、保存処理は同期処理として`Effect<None, Never>`を返却しています。

```swift
public struct FirebaseAPIClient {
    // Auth
    public var checkUserStatus: () -> Effect<AppUser, Never>
    public var signInAnonymously: () -> Effect<None, APIError>
    public var signUp: (_ email: String, _ password: String) -> Effect<None, APIError>

    // Pomodoro History
    public var savePomodoroHistory: (_ history: PomodoroTimerHistory) -> Effect<None, Never>
    
    ...
}
```



## おわりに

今回はシンプルなアプリであったためTCAを使うことによりコード量が増加した感じがあります。
しかし、TCAにおけるステートの管理方法やReducerの分割、統合の仕方を学べることができました。

誰かの参考になれば幸いです。
