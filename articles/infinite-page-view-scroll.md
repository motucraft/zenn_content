---
title: "【Flutter】PageViewを使ってなるべく簡単に無限循環スクロールを模倣する"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "PageView"]
published: false
---

# 1. はじめに

無限に循環スクロールするWidgetが欲しくなりました。
[indexed_list_view](https://pub.dev/packages/indexed_list_view)を見つけたので、これで実現できるかもと思ったのですが、

しかし、以下のissueにもあるとおり、残念ながら[FixedExtentScrollPhysics](https://api.flutter.dev/flutter/widgets/FixedExtentScrollPhysics-class.html)や[PageScrollPhysics](https://api.flutter.dev/flutter/widgets/PageScrollPhysics-class.html)などのScrollPhysicsを使用できないようです。

https://github.com/marcglasberg/indexed_list_view/issues/36

スナップを効かせてスクロールするWidgetが中央に配置されるようにしたいため、ScrollPhysicsを拡張して試行錯誤してみたのですが、負のインデックスと正のインデックスの境界のスクロールを上手く制御できず諦めました...

そのため、もっと簡単に無限に循環するスクロールを模倣できないものかと考えました。

# 2. 挙動

循環の境界部分において、少し滑らかさを欠いているような気がします。ただまぁ模倣ですので許容範囲でしょうか。

![](https://storage.googleapis.com/zenn-user-upload/36aceaabce63-20240204.gif)

# 3. コード

コードはこちら。

https://github.com/motucraft/descending_app_bar

これだけです（hooksをを使用しています）。

```dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Infinite PageView Scroll',
      theme: ThemeData.dark(),
      home: const InfinitePageViewScroll(),
    );
  }
}

class InfinitePageViewScroll extends HookWidget {
  static final List<String> urls = [
    'https://p.npb.jp/img/common/logo/2024/logo_b_l.gif',
    'https://p.npb.jp/img/common/logo/2024/logo_m_l.gif',
    'https://p.npb.jp/img/common/logo/2024/logo_h_l.gif',
  ];

  const InfinitePageViewScroll({super.key});

  @override
  Widget build(BuildContext context) {
    final controller =
        usePageController(viewportFraction: 0.75, initialPage: 1);

    final itemCount = urls.length + 2;

    useEffect(() {
      controller.addListener(() {
        if (controller.page! > (urls.length + 0.999)) {
          controller.jumpToPage(1);
        } else if (controller.page! < 0.001) {
          controller.jumpToPage(itemCount - 2);
        }
      });
      return null;
    }, []);

    return Scaffold(
      appBar: AppBar(title: const Text('Infinite PageView Scroll')),
      body: PageView.builder(
        controller: controller,
        itemBuilder: (context, index) {
          if (index == 0) {
            return Image.network(urls.last);
          } else if (index == itemCount - 1) {
            return Image.network(urls.first);
          } else {
            return Image.network(urls[(index - 1) % urls.length]);
          }
        },
      ),
    );
  }
}
```

# 4. おわりに

当初は`PageScrollPhysics`の`_getTargetPixels`をカスタムすればいけそうだと思ったのですが、`ScrollPhysics`をかなり深く理解しないと無理だなと思いました。
これに限らずですが、普段使っているSDKのWidgetやクラス群の中身をもっと理解しないとダメだなと思います。（でも難しいんだよなー）
