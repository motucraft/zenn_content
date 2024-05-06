---
title: "【Flutter】画像を含んだWidgetを"もっと見る"で折り畳み開閉可能とする" (See More/Read More)"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "flutter_hooks", "cached_network_image"]
published: false
---

# 1. はじめに

長いテキストやWidgetを折り畳みたい場合のサンプルです。

以下のパッケージがありましたが、このパッケージはテキストにしか適用できないようですね。

https://pub.dev/packages/readmore

テキストだけでなく画像が入っているようなケースでも折り畳んで、「もっと見る」をタップしたら全体を表示するようにしたいです。

# 2. 挙動

以下のような挙動になりました。`See More`をタップすると全体を表示し、`Close`タップで閉じます。画像はネットワークから入手しています（[Lorem Picsum](https://picsum.photos/)を使用させていただきました）。

![](https://storage.googleapis.com/zenn-user-upload/b9f1b7f519a0-20240506.gif =500x)

# 3. コード

以下の`ExpandableSeeMore`クラスが中心となるコードです。

https://github.com/motucraft/expandable_see_more/blob/main/lib/main.dart#L114-L214

```dart
useEffect(() {
  () async {
    if (completer != null) {
      await completer;
    }

    WidgetsBinding.instance.addPostFrameCallback((_) {
      final box = contentKey.currentContext?.findRenderObject() as RenderBox?;
      if (box != null && box.size.height < collapsedHeight) {
        isExpandable.value = false;
      }
    });
  }();

  return null;
}, [child]);
```

このuseEffectフック内で対象Widgetのサイズを入手するようにしています。画像はネットワークから入手しますので、画像のロード完了後にサイズを取得しなければならないため、Completerを使って遅延させています。

あとは、`See More`タップ/`Close`タップを[flutter_hooks](https://pub.dev/packages/flutter_hooks)の`useState`で状態管理しつつ、`AnimatedSize`や`ConstrainedBox`で表示領域を変更しているだけです。

```dart
final imageLoadCompleter = Completer();
final image = CachedNetworkImage(
  imageUrl: 'https://picsum.photos/id/197/800/600',
  imageBuilder: (_, imageProvider) {
    if (!imageLoadCompleter.isCompleted) {
      imageLoadCompleter.complete();
    }

    return Image(fit: BoxFit.fitWidth, image: imageProvider);
  },
);
return ExpandableSeeMore(
  completer: imageLoadCompleter.future,
  child: Wrap(
    alignment: WrapAlignment.center,
    children: [
      Text('Some Text. ' * 10),
      image,
      const Text('No Baseball, No Life.', style: TextStyle(fontSize: 24)),
    ],
  ),
);
```

`ExpandableSeeMore`の呼び出し元は、上記のようにCompleterとWidgetを渡しています。

# 4. おわりに

画像を含んだWidgetを折り畳んだり展開したりして表示することができました。

`See More`で展開したときにスクロール位置を調整して、当該コンテンツがトップに表示されるようにできたりすると良いかもとも思いましたが、最低限こんなところでしょうか。

あとは、`ExpandableSeeMore`のchildへ例えばColumnウィジェットを渡すとおそらくオーバーフローすることになると思いますので、この辺り改善点かなと思います。
