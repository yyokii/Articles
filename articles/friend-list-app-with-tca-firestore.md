---
title: "[iOS] TCA × Firestore で友達リストアプリを作った"
emoji: "🪢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ios, swiftui, tca, firebase, firestore]
published: true
---

## 何をつくったか

人物の情報を管理できるアプリを作りました。

グループも作成でき、例えば「友達」や「家族」などのグループで整理することができます。

![スクリーンショット](/images/friend-list-app-with-tca-firestore/app-screen-shot.png)

https://apps.apple.com/jp/app/id6475261189

ソースコードはこちら
https://github.com/yyokii/RelNet

## なぜつくったの

1つは TCA のキャッチアップです。業務で利用しているので勉強として作成しました。

もう1つは自分が欲しかったからです。

人の情報をみなさんはどう管理していますか？

自分は「メモ」アプリに書いたり「Trello」に書いたりで、まとまって管理することができていませんでした。まとめられていないとふと見るときに不便なのと、書く意欲もあまり起きませんでした。

iOS には標準の「連絡先」アプリがありますが、「連絡先」というアプリ名の通り連絡先を管理することに特化したアプリになっています。

* 連絡先（電話番号やSlack）を知らないけどたまに会う人
* 誕生日や趣味、好きな食べ物などその人に関連する情報

この情報をまとめられると幸せだなと思いアプリを作りました。

## 利用したもの

### Teck Stacks

* SwiftUI
* The Composable Architecture
* Xcode Cloud
* Firebase
  * Firebase Auth
  * Firestore
  * Firebase Crashlytics
  * Cloud Functions for Firebase

特に変わったものは採用しておらずよくあるものだと思います。
バックエンドは、慣れていることもあり Firebase を使用しています。

### デザイン

* SF Symbols
* ChatGPT
* Clipdrop

デザインを凝ると作成目的からずれてしまう（デザイン実現に時間がかかってしまう）のと、そこに拘ってうまくいった試しがないのでシンプルなデザインにしました。

iOSの基本的なコンポーネント/色を使用しアイコンが必要な場合は SF Symbols を使用しています。

アプリアイコンは ChatGPT でアイデアをもらいながら DALL-E で作成し Clipdrop で拡張し、一部手を加えて作成しました。生成AIがあることでぱっとそれっぽいアイコンができるのはありがたいですね。

## 設計

TCAです。

マルチモジュールにはしていません。
個人開発且つ小さいプロジェクトなので、マルチモジュールにすることによる恩恵があまりなさそうと思いこうしました。

フォルダ構成は下記のようになっています。

![フォルダ構成](/images/friend-list-app-with-tca-firestore/folders.png)

## TCA

TCA のリポジトリにある Examples の [SyncUps](https://github.com/pointfreeco/swift-composable-architecture/tree/main/Examples/SyncUps) を参考にして作成しました。

README にもあるように、一般的なアプリで使用されるであろうものが網羅されており非常に参考になりました。

> This project demonstrates how to build a complex, real world application that deals with many forms of navigation (*e.g.*, sheets, drill-downs, alerts), many side effects (timers, speech recognizer, data persistence), and do so in a way that is testable and modular.
>
> （訳）
>
> *このプロジェクトでは、さまざまな形式のナビゲーション (シート*、ドリルダウン、アラートなど)、多くの副作用 (タイマー、音声認識、データ永続化) を処理する複雑な今日のアプリを構築する方法を示します。また、テスト可能でモジュール化されています。

使用している TCA のバージョンは現状 1.6.0 です。

バージョン 1.4 で @Reducer macro が導入され、1.5 での更新も併せると、
今までよりキーパスでシンプルにアクセスできるようになっています。

例えば以下のように変更が可能になります

Scope 

```diff swift
-       Scope(state: \.login, action: /Action.login) {
+       Scope(state: \.login, action: \.login) {
            Login()
        }
```

ifLet

```diff swift
-       .ifLet(\.$alert, action: /Action.alert)
+       .ifLet(\.$alert, action: \.alert)
```

navigationDestination

```diff swift
       .navigationDestination(
-           store: store.scope(state: \.$destination, action: Main.Action.destination),
-           state: /Main.Destination.State.personDetail,
-           action: Main.Destination.Action.personDetail
+           store: store.scope(state: \.$destination.personDetail, action: \.destination.personDetail)
        ) {
            PersonDetailView(store: $0)
        }
```

また、`TaskResult` も Deprecated となっています。
詳細はこちら:  [Moving off of TaskResultin page link](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.4/#Moving-off-of-TaskResult)

### TCA と Firestore

Firestore のデータ操作と Firebase Auth での認証処理用に Dependency を作成しています。

データを listen するものについては `AsyncThrowingStream` を返すようにしています。

参考: 
 https://zenn.dev/zones/articles/5f845b065e64f5

## SwiftUI

グループ一覧を表示する箇所はいわゆるフローレイアウトになっているのですが、これは Apple のサンプルコードのものを使用しています。
[Food Truck: Building a SwiftUI multiplatform app | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/food_truck_building_a_swiftui_multiplatform_app/)

![フローレイアウト](/images/friend-list-app-with-tca-firestore/group-layout.png)

また、String Ctalog を使用しています。
導入が簡単でサクッとローカライズできていいですね。

String Catalog のキー名については、

* モジュール分割していないので、コンテキストがわかるような命名をして重複が生じないように
* ローカライズされてることがわかるように "login-alert-title" のような形式で統一

としました。

![String Catalog](/images/friend-list-app-with-tca-firestore/string-catalog.png)

## 今後

AI機能を追加しようと思っています。

+ 入力情報を元にプロファイリングしてくれる
+ 入力情報を元にトークトピックやメッセージ内容を提案してくれる

のような機能があると面白そうかなと思っています。

このアプリが、日々の出会いをよりワクワクするためになれば幸いです。

https://apps.apple.com/jp/app/id6475261189

https://github.com/yyokii/RelNet
