---
title: "【Flutter】現在位置から画面中央へgoogle_maps_flutterのPolylineを引く"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "google_maps_flutter"]
published: true
---

# 1. はじめに

タクシーアプリなどで見かけるアレです。[google_maps_flutter](https://pub.dev/packages/google_maps_flutter) でやってみます。

|                                     Polylineを引く                                      |                                  zoom level 変更時の挙動                                   |
|:------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------:|
| ![](https://storage.googleapis.com/zenn-user-upload/83d224521ed2-20250209.gif =300x) | ![](https://storage.googleapis.com/zenn-user-upload/eafec18f8fc2-20250209.gif =300x) |

https://github.com/motucraft/google_maps_flutter_playground/blob/master/lib/current_location_polyline.dart

# 2. 困ったこと

## 2.1 パフォーマンス劣化

直線の [Polyline](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/Polyline-class.html) であれば良いのですが、破線を描画しようとするとzoom levelによってマップのパフォーマンスが著しく劣化する事象に困りました。パフォーマンス劣化して最終的にはアプリがダウンします。ズームアウトさせた場合に顕著でしたので、破線の描画数が膨大になることが問題なのだろうと考えました。

そこで、ズームアウトした(zoom levelが小さくなった)場合は、以下のように指数関数的に [PatternItem](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/PatternItem-class.html) の `length` を変化させるようにして回避しました。

https://github.com/motucraft/google_maps_flutter_playground/blob/67a1bb37f5783ac8557846137266c1a79cae7d4d/lib/current_location_polyline.dart#L215-L224

## 2.2 Current Position

[geolocator](https://pub.dev/packages/geolocator) で現在位置を入手しています。 しかし、`google_maps_flutter` がデフォルトで表示してくれる現在位置を示す青い丸と、`geolocator` が返してくる位置情報が完全一致しません。

これは位置情報の取得方法が違うのだろうなと思うのですが、`google_maps_flutter` は位置情報を返してくれるインタフェースはありません。そのため、`Polyline` の端を現在位置にピッタリフィットさせることができません。

今回はスルーしましたが、ここにこだわる場合は `google_maps_flutter` のデフォルトの現在位置アイコンの表示をやめて自前で表示するしかなさそうですね。

# 3. おわりに

[google_maps_flutter_platform_interface](https://pub.dev/packages/google_maps_flutter_platform_interface) の [Changelog](https://pub.dev/packages/google_maps_flutter_platform_interface/changelog#2100) を見ていると、`2.10.0` で `Adds support for ground overlay.` とありますね。

以前からFeature Requestが出ていたグラウンドオーバーレイがようやくサポートされそうです。

https://github.com/flutter/flutter/issues/26479

現時点では、まだ本体のPRはマージされておらず、サブPRがマージされたという状況のようです。

https://github.com/flutter/packages/pull/8518

https://github.com/flutter/packages/pull/8432

対応されることは無いのかなと思っていましたが、進捗しているようで何よりです。動向を気にしておこうと思います。
