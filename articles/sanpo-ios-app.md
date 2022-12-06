---
title: "[iOS] iOS16機能の素振りでシンプル歩数計アプリをつくった"
emoji: "🚶‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ios, swift, swiftUI, ios16]
published: true
---

:::message 
これは [ZOZO Advent Calendar 2022](https://qiita.com/advent-calendar/2022/zozo) カレンダー Vol.1 の 7 日目の記事です。
:::

## 何をつくったか

シンプルな歩数計アプリを作ってみました。

![sanpo app](/images/sanpo-app.png)

ソースコードはこちら
https://github.com/yyokii/Sanpo

開発目的は主に以下です。

* iOS16のロック画面のウィジェットを使ってみたかった
* UICalendarViewを利用してみたかった
* シンプルにViewとModel（データ+ロジック）に分離した設計でアプリを作ってみたかった

## 利用したiOS16~のもの

2022年9月13日（JST）にiOS16がリリースされ、いくつか新しい機能、APIが提供されるようになりました。今回のアプリではその一部を利用しています。

### WidgetKit（ロック画面のウィジェット）

![lock screen widget](/images/sanpo-lock-screen-widget.png)

ホームウィジェットの表示は以下のように`supportedFamilies(_:)`を追加することで実現可能です。

```swift
import WidgetKit
import SwiftUI

@main
struct SanpoWidget: Widget {
    let kind: String = "Widget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            WidgetEntryView(entry: entry)
        }
        .configurationDisplayName("Sanpo")
        .supportedFamilies([.accessoryCircular])
    }
}
```

### WeatherKit

WeatherKitは気象データにアクセスできるフレームワークです。

> WeatherKitを使用して幅広いデータに基づく有益な気象情報をAppやサービスで提供することにより、ユーザーが最新情報を確認し、身を守り、備えるのをサポートすることができます。iOS 16、iPadOS 16、macOS 13、tvOS 16、watchOS 9向けにはプラットフォーム固有のSwift APIを使用し、その他すべてのプラットフォーム向けにはREST APIを使用することで、AppでWeatherKitを簡単に利用できます。

（引用）[WeatherKitの概要 - Apple Developer](https://developer.apple.com/jp/weatherkit/)

今回は現在〜6時間後までの1時間ごとの天気を表示するために利用しました。

下記のように、`WeatherService`オブジェクトを介してリクエストすることで対象のデータ（今回の場合は毎時のデータ）が取得できます。

```swift
@MainActor
public class WeatherData: ObservableObject {

~~~

    @Published public var phase: AsyncStatePhase = .initial
    @Published public var hourlyForecasts: Forecast<HourWeather>?
    private let service = WeatherService.shared

~~~
  
    private func loadHourlyForecast(for location: CLLocation) async {
        phase = .loading
        let hourWeather = await Task.detached(priority: .userInitiated) {
            let forecast = try? await self.service.weather(
                for: location,
                including: .hourly
            )
            return forecast
        }.value
        hourlyForecasts = hourWeather
        phase = .success(Date())
    }
}
```

各データの表示については下記のように先頭から任意の数を取り出し、

```swift
if let hourlyForecasts = weatherData.hourlyForecasts {
  ForEach(hourlyForecasts[0..<6], id: \.self.date) { forecast in
                                                    weatherDataRow(
                                                      date: forecast.date,
                                                      weatherIconName: forecast.symbolName,
                                                      temperature: forecast.temperature,
                                                      precipitationChance: forecast.precipitationChance
                                                    )
                                                   }
  .padding(.bottom, 8)
}
```
個々のデータについては`Date.FormatStyle()`や`formatted(_:)`を用いて表示用データに変換しています。

```swift
func weatherDataRow(
  date: Date,
  weatherIconName: String,
  temperature: Measurement<UnitTemperature>,
  precipitationChance: Double
) -> some View {
  HStack {
    Text(date, format: Date.FormatStyle().hour(.defaultDigits(amPM: .abbreviated)).minute())
    .adaptiveFont(.normal, size: 18)
    Spacer()
    HStack {
      Image(systemName: weatherIconName)
      Text(temperature.formatted(.measurement(width: .abbreviated, usage: .weather)))
      .adaptiveFont(.normal, size: 18)
      .frame(width: 90, alignment: .leading)
    }
    Spacer()
    Text(formattedPrecipitationChance(precipitationChance))
    .adaptiveFont(.normal, size: 18)
  }
}
```

データ取得とその表示についてはAppleのサンプルコードもあり、そちらを参考にしています。

Fetching weather forecasts with WeatherKit | Apple Developer Documentation](https://developer.apple.com/documentation/weatherkit/fetching_weather_forecasts_with_weatherkit)

また、WeatherKitを利用する場合は「Apple Weatherの商標」と、「[データソース](https://weatherkit.apple.com/legal-attribution.html)への法的リンク」の表示が必要です。

> AppでApple Weatherのデータを表示する場合は、[WeatherKitドキュメント](https://developer.apple.com/jp/weatherkit/get-started/index.html#attribution-requirements)のアトリビューション要件に従う必要があります。

（引用）[App Store Reviewガイドライン - Apple Developer](https://developer.apple.com/jp/app-store/review/guidelines/#intellectual-property)

### UICalendarView

iOS16より日付の選択、表示変更が可能なカレンダークラスである`UICalendarView`が利用できます。
今回のアプリでは過去の歩数データを表示するために利用しています。

![sanpo app](/images/sanpo-calendar-view.png)

[UICalendarView | Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uicalendarview)

## 設計

ViewとModel（Entityとロジックをもつもの）のレイヤーを持つシンプルなものです。

* 歩数などのモデルについてはActive Recordパターンを採用し、ロジックとFactoryメソッドを持ちます。
* 状態管理などのライフサイクルが必要となる場合は `~Data`という命名のclassを作成しています。

[Stop using MVVM for SwiftUI | Apple Developer Forums](https://developer.apple.com/forums/thread/699003?page=1) での議論を読みその簡易な実践の意味もあり、MVの設計としています。
そこでの議論ではSwiftの機能を利用し設計をシンプルに保つことを重要視していますが、それは「A Philosophy of Software Design」での内容と通じることが多いように思いました。
[「A Philosophy of Software Design」のまとめと感想](https://zenn.dev/yyokii/articles/book-a-philosophy-of-software-design)

## おわりに

SwiftUIの登場から数年が経ちますが、Swiftの言語機能の進化に加えて毎年OSの更新があり常に考えることが絶えませんね。そんな訳でついインプットが多くなりがちですが、「触ってみる」ことが「使う」ためには近道です。

シンプルなアプリですが、それを再認識できて良かったです。
誰かの参考になれば幸いです。
