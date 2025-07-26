---
title: "【Flutter】google_maps_flutter で矢印を引く"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "google_maps_flutter"]
published: true
---

# 1. はじめに

[google_maps_flutter](https://pub.dev/packages/google_maps_flutter) を使って、マップ上に矢印を引いてみようと思います。このような挙動です。

|                                         作成                                         |                                         追加                                         |                                          削除                                          |
|:------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------:|
| ![](https://storage.googleapis.com/zenn-user-upload/f6067ddff1e3-20250726.gif =200x) | ![](https://storage.googleapis.com/zenn-user-upload/db82b4961cd6-20250726.gif =200x) | ![](https://storage.googleapis.com/zenn-user-upload/79ff4dd78c99-20250726.gif =200x) |

マップ上にラインを引くためには、[Polyline](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/Polyline-class.html) を使用します。`Polyline` クラスのプロパティには、[startCap](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/Polyline/startCap.html)、[endCap](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/Polyline/endCap.html) がありますから、当初はこれを使用すれば矢印の先端を描画できると考えました。
しかしこれは、Android では問題なく動作しましたが、iOS では機能しないようでした。リファレンスにも `Not supported on all platforms.` と記載があります。

そして、Issueを漁ってみると [stuartmorgan-g](https://github.com/stuartmorgan-g) さんもこのように言っています。

https://github.com/flutter/flutter/issues/161950#issuecomment-2605319757

> The Google Maps SDK on iOS does not have these features, so the plugin cannot support them. You'd need to direct this feature request to the Google Maps team. All the plugin can do is document the underlying SDK limitation (here, here, and here).

この制約を回避するため、`Polyline` とは別に [Marker](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/Marker-class.html) を用いて矢印の先端を実装することにしました。

# 2. コード

コードの全量はこちらです。

https://github.com/motucraft/google_maps_flutter_playground/blob/master/lib/arrow_with_polyline.dart

# 3. 矢印の先端描画

矢印の先端は、`Polyline` の最後の地点に `Marker` を配置することで実現しています。
各 `Polyline` の points の長さが2以上であることを確認し、最後の2点 prev と last 間の方位角を [Geolocator.bearingBetween](https://pub.dev/documentation/geolocator/latest/geolocator/Geolocator/bearingBetween.html) で計算後、0〜360度に正規化した rotation を `Marker` に設定しています。

```dart
@riverpod
class ArrowMarkersNotifier extends _$ArrowMarkersNotifier {
  @override
  Set<Marker> build() {
    final Set<Polyline> polylines = ref.watch(arrowPolylinesNotifierProvider);
    final icons = ref.watch(arrowIconProvider).valueOrNull;
    if (icons == null) {
      return {};
    }

    final (:chevronUpRed, :chevronUpBlue) = icons;

    ref.listen(highlightedPolylineIdNotifierProvider, (_, next) {
      if (next != null) {
        state =
            state.map((marker) {
              if (marker.markerId.value == next) {
                return marker.copyWith(iconParam: chevronUpBlue);
              }
              return marker.copyWith(iconParam: chevronUpRed);
            }).toSet();
      }
    });

    return polylines.where((p) => p.points.length >= 2).map((polyline) {
      final pts = polyline.points;
      final prev = pts[pts.length - 2];
      final last = pts.last;
      final rawBearing = Geolocator.bearingBetween(
        prev.latitude,
        prev.longitude,
        last.latitude,
        last.longitude,
      );
      final rotation = (rawBearing + 360) % 360;

      return Marker(
        markerId: MarkerId(polyline.polylineId.value),
        position: last,
        icon: chevronUpRed,
        rotation: rotation,
        anchor: const Offset(0.5, 0.5),
        consumeTapEvents: true,
        onTap: () {
          ref
              .read(highlightedPolylineIdNotifierProvider.notifier)
              .select(polyline.polylineId.value);
        },
      );
    }).toSet();
  }
}
```

# 4. おわりに

最初は [Canvas](https://api.flutter.dev/flutter/dart-ui/Canvas-class.html) 上に矢印を描いて、それをマップへ落とす ([GroundOverlay](https://pub.dev/documentation/google_maps_flutter/latest/google_maps_flutter/GroundOverlay-class.html)を使うなどして) というイメージで考えていたのですが、Canvas→Bitmap 変換やスクリーン座標と緯度経度の対応づけなど複雑化しそうでした。そのため、今回は `Polyline` と `Marker` の組み合わせで矢印を描画する手法を採用しました。

今回のサンプルでは削除しかできないのですが、矢印を再編集できると良いなと思っています。
