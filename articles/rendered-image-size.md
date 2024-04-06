---
title: "【Flutter】レンダリング後のImage.asset画像の描画サイズが取れなくてハマる"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart"]
published: true
---

# 1. はじめに

[Image.asset](https://api.flutter.dev/flutter/widgets/Image/Image.asset.html)で画像を表示して、実際に描画されたWidgetのサイズを取得したかったのですが、プロセス起動後の初回だけ画像のサイズが0で取れてしまうという問題にハマりました。

StackOverflowです。同じ問題にハマっている人がいました。
https://stackoverflow.com/questions/68880220/flutter-image-asset-actual-size-issue

# 2. NG

最初は、フレームの描画処理が終わってからサイズを取れば良いので`WidgetsBinding.instance.addPostFrameCallback`を使えば良いのでは？と考えました。
しかし、以下のコードでは正しくサイズが取れません。

（実際はもっと複雑で画面遷移が発生する状況で悩みました。プロセス起動後に一度でも読み込み済みで、再度表示するようなケースでは問題が発生せず、プロセス起動後初回のみというところでハマりました。）

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: RenderedImageSize(),
    );
  }
}

class RenderedImageSize extends StatefulWidget {
  const RenderedImageSize({super.key});

  @override
  State<RenderedImageSize> createState() => _RenderedImageSizeState();
}

class _RenderedImageSizeState extends State<RenderedImageSize> {
  final _imageKey = GlobalKey();
  Size? _imageSize;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      final renderBox =
          _imageKey.currentContext?.findRenderObject() as RenderBox?;
      if (_imageSize == null && renderBox != null) {
        setState(() {
          _imageSize = renderBox.size;
        });
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    // build内でもダメ
    // WidgetsBinding.instance.addPostFrameCallback((_) {
    //   final renderBox =
    //       _imageKey.currentContext?.findRenderObject() as RenderBox?;
    //   if (_imageSize == null && renderBox != null) {
    //     setState(() {
    //       _imageSize = renderBox.size;
    //     });
    //   }
    // });

    return Scaffold(
      appBar: AppBar(title: const Text('Rendered Image Size')),
      body: Column(
        children: [
          Image.asset(
            key: _imageKey,
            'assets/image.webp',
          ),
          const SizedBox(height: 16),
          Text(
            'width:${_imageSize?.width.toStringAsFixed(1)}, height:${_imageSize?.height.toStringAsFixed(1)}',
            style: const TextStyle(fontSize: 24),
          ),
        ],
      ),
    );
  }
}
```

このように、サイズが0.0で取れてしまいます。
![](https://storage.googleapis.com/zenn-user-upload/6b156528cb38-20240406.png =200x)

画像フレームがまだレンダリングされていないのに、その前にサイズを取ってしまうからでしょうか？
`Future.delayed(const Duration(milliseconds: 500), () {})`みたいなことをして、サイズ取得の処理を遅延させれば上手くいきますがそんなことはしたくありません。どれだけ遅延させれば良いかも分かりませんし。

[ImageFrameBuilder](https://api.flutter.dev/flutter/widgets/ImageFrameBuilder.html)には以下の記述があります。

> The frame argument specifies the index of the current image frame being rendered. It will be null before the first image frame is ready, and zero for the first image frame. For single-frame images, it will never be greater than zero. For multi-frame images (such as animated GIFs), it will increase by one every time a new image frame is shown (including when the image animates in a loop).

私が使っている画像はsingle-frame imageなんですけど...う〜ん

# 3. OK

`ImageFrameBuilder`を使って、コールバック内でサイズを取得するように変更すると、上手くサイズが取得できました。

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: RenderedImageSize(),
    );
  }
}

class RenderedImageSize extends StatefulWidget {
  const RenderedImageSize({super.key});

  @override
  State<RenderedImageSize> createState() => _RenderedImageSizeState();
}

class _RenderedImageSizeState extends State<RenderedImageSize> {
  final _imageKey = GlobalKey();
  Size? _imageSize;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Rendered Image Size')),
      body: Column(
        children: [
          Image.asset(
            key: _imageKey,
            'assets/image.webp',
            frameBuilder: (_, child, frame, __) {
              if (frame != null) {
                Future.microtask(() {
                  final renderBox = _imageKey.currentContext?.findRenderObject()
                      as RenderBox?;
                  if (_imageSize == null && renderBox != null) {
                    setState(() {
                      _imageSize = renderBox.size;
                    });
                  }
                });
              }
              return child;
            },
          ),
          const SizedBox(height: 16),
          Text(
            'width:${_imageSize?.width.toStringAsFixed(1)}, height:${_imageSize?.height.toStringAsFixed(1)}',
            style: const TextStyle(fontSize: 24),
          ),
        ],
      ),
    );
  }
}
```

レンダリングされたWidgetのサイズが表示されています。

![](https://storage.googleapis.com/zenn-user-upload/17cac5fa827b-20240406.png =500x)

# 4. OK (with flutter_hooks)

[flutter_hooks](https://pub.dev/packages/flutter_hooks)を使う場合はこのような感じでしょうか。やっていることは同じです。

```dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: RenderedImageSize(),
    );
  }
}

class RenderedImageSize extends HookWidget {
  const RenderedImageSize({super.key});

  @override
  Widget build(BuildContext context) {
    final imageKey = useMemoized(() => GlobalKey());
    final imageSize = useState<Size?>(null);

    return Scaffold(
      appBar: AppBar(
        title: const FittedBox(
            child: Text('Rendered Image Size with flutter_hooks')),
      ),
      body: Column(
        children: [
          Image.asset(
            key: imageKey,
            'assets/image.webp',
            frameBuilder: (_, child, frame, __) {
              if (frame != null) {
                Future.microtask(() {
                  final renderBox =
                      imageKey.currentContext?.findRenderObject() as RenderBox?;
                  if (imageSize.value == null && renderBox != null) {
                    imageSize.value = renderBox.size;
                  }
                });
              }
              return child;
            },
          ),
          const SizedBox(height: 16),
          Text(
            'width:${imageSize.value?.width.toStringAsFixed(1)}, height:${imageSize.value?.height.toStringAsFixed(1)}',
            style: const TextStyle(fontSize: 24),
          ),
        ],
      ),
    );
  }
}
```

こちらも問題なくレンダリング後のWidgetサイズを取得できました。

![](https://storage.googleapis.com/zenn-user-upload/800e42b5b74c-20240406.png =500x)

# 5. おわりに

未だにこういう問題にハマるのは、結局表面的な理解しかできていないからなんですよね。
もっと内部の挙動を把握しないと。やっぱり、SDK内部のコード読めよってことですよね...
