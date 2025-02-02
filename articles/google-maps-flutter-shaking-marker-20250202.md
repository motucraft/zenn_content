---
title: "【Flutter】google_maps_flutterのMarkerをプルプルさせる"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "google_maps_flutter"]
published: true
---

# 1. はじめに

[google_maps_flutter](https://pub.dev/packages/google_maps_flutter) のマーカーをプルプル震わせてみます。

![](https://storage.googleapis.com/zenn-user-upload/3ff0866d0ebb-20250202.gif =300x)

https://gist.github.com/motucraft/12b767327c5bc039e6f2ac3dd779926a

# 2. 悩み

`google_maps_flutter` の [Marker](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/Marker-class.html) 周りを触るとき、いつも厄介だなと思うのが [BitmapDescriptor](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/BitmapDescriptor-class.html) ですね。これはもう何年も前から...

- [[google_maps_flutter] Support Widgets as markers](https://github.com/flutter/flutter/issues/24213)
- [Maps SDK for Flutter (IssueTracker)](https://issuetracker.google.com/issues/140415468)

私が最初にこの議論を認識したのは3, 4年前だったと思います。エコシステムの成長とともに「そのうち解消されるっしょ」と思っていましたが、状況は変わらないまま既に2025年を迎えました。この先もおそらく変わらないでしょうね😢

他の選択肢としては、[flutter_map](https://pub.dev/packages/flutter_map) を使うという選択もあると思います。
ただ、Mapの情報量としてはやはり Google Map が圧倒的で、なかなか [OpenStreetMap](https://www.openstreetmap.org/#map=19/35.211540/136.840856&layers=P)、[Mapbox](https://www.mapbox.jp/) など他のタイルベンダーを使用するという選択に至りません。

他には、`flutter_map` を利用しつつタイルレイヤに Google Map の [Map Tiles API](https://developers.google.com/maps/documentation/tile?hl=ja) を使うという方法もあると思いますが、

https://docs.fleaflet.dev/tile-servers/using-google-maps
> Tile providers that also provide their own SDK solution to display tiles will often charge a higher price to use 3rd party libraries like flutter_map to display their tiles. Just another reason to switch to an alternative provider.

ということで、なかなか現実的ではなさそうに思っています。[Map Tiles API の使用量と請求額](https://developers.google.com/maps/documentation/tile/usage-and-billing?hl=ja) を見ると、1タイルのリクエストごとに課金されそうなので費用面が怖いですよね。

## 2.1 怪しそうなアイデア

そのような中で、こんな手もあるのか！！と思ったのがこれです。
- [Using 'google_maps_flutter' tiles with 'flutter_map'](https://github.com/fleaflet/flutter_map/discussions/1158)
- [google_tile_layer_widget.dart](https://gist.github.com/Ahmed-gubara/2d191bcf3e426521f85f5bf44a071cd7)

`flutter_map` のタイルレイヤに `google_maps_flutter` を使うというアイデアですね。
[GoogleMapの方はIgnorePointer](https://gist.github.com/Ahmed-gubara/2d191bcf3e426521f85f5bf44a071cd7#file-google_tile_layer_widget-dart-L59) して、[mapEvent](https://gist.github.com/Ahmed-gubara/2d191bcf3e426521f85f5bf44a071cd7#file-google_tile_layer_widget-dart-L40) にて `flutter_map` と `google_maps_flutter` の同期をかけているようです。

この方法は、`google_maps_flutter` を使用していることにはなるので Google Map の利用規約違反にはならない、という論理なのでしょうか？怪しいと思いますけどね..
機能的にも [こちらのコメント](https://gist.github.com/Ahmed-gubara/ebfb48dbdbff3dc7051ff0998d3e7048?permalink_comment_id=5056919#gistcomment-5056919) にあるようにAndroidでマーカーが遅延する問題があるようでした。`flutter_map` と `google_maps_flutter` の同期をかけるという仕組みなので、このような問題も発生するのでしょうね。

# 2.2 [map](https://pub.dev/packages/map) パッケージ

[map](https://pub.dev/packages/map) というパッケージもあるようです。
これも事情は同じように思います。exampleを見てみると、以下のようにタイルサーバを利用しており、`Legal notice` と書かれています。これは Google Map の利用規約に違反すると思いますので商用利用には難しそうですね。

https://github.com/xclud/flutter_map/blob/d7979cdca4c25369f60c4539473562edaa62d47e/example/lib/utils/tile_servers.dart#L3-L10

---
色々と考えてはみるものの、結局 `google_maps_flutter` を利用するしかなさそう、という悩みでした。

# 3. おわりに

2025年にもなったので、`google_maps_flutter` の状況変化していないかな？と少し探ってみたのですが、悩みは変わらずでした。Googleさんのやる気はなさそうですので、この先も変わらないのだろうなと思っています。

有名どころのアプリ（ナビアプリ、タクシー配車アプリ、宅配アプリなど）を眺めてみても、どれもネイティブ実装のようですね。マップが機能の中心だったり存在価値となるようなアプリを Flutter で、というのは苦しいでしょうね。もちろん、ROI次第でしょうけど。
