---
title: "【Flutter】LicenseRegistryを使って自前でライセンスページを表示する"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "showLicensePage"]
published: false
---

# 1. はじめに

Flutterでライセンスページを用意したい場合、SDK標準で簡単な方法が用意されています。

https://docs.flutter.dev/resources/faq#how-can-i-determine-the-licenses-my-flutter-application-needs-to-show

> There’s an API to find the list of licenses you need to show:
> - If your application has a Drawer, add an AboutListTile.
> - If your application doesn’t have a Drawer but does use the Material Components library, call either showAboutDialog or showLicensePage.
> - For a more custom approach, you can get the raw licenses from the LicenseRegistry.

確かに、[AboutListTile](https://api.flutter.dev/flutter/material/AboutListTile-class.html)や[showLicensePage](https://api.flutter.dev/flutter/material/showLicensePage.html)を利用すれば非常に簡単です。
例えばこれだけで済みます。

```dart
ElevatedButton(
  child: const Text('showLicensePage'),
  onPressed: () => showLicensePage(context: context),
)
```

![](https://storage.googleapis.com/zenn-user-upload/18f68529c6b3-20240307.png =200x)

でもこれ、カスタマイズできないんですよね。`Powered by Flutter`が必ず表示されますし。
そこで、ライセンスページを自前で用意して見ようと思います。

# 2. 例

色などは適当ですが、このように表示できました。

||     |
|:---:|:---:|
|![](https://storage.googleapis.com/zenn-user-upload/7270f6eb1155-20240307.png)|![](https://storage.googleapis.com/zenn-user-upload/a33cd7d01b2c-20240307.gif)|

# 3. コード

> - For a more custom approach, you can get the raw licenses from the LicenseRegistry.

ドキュメントにこのように記載されていますので、[LicenseRegistry](https://api.flutter.dev/flutter/foundation/LicenseRegistry-class.html)を使用します。

パッケージ名を入手するのは簡単そうなのですが、paragraphをどのように処理すれば良いのか悩みました。インデントも考慮しないと上記のような見た目にはなりません。
そこで、[showLicensePage](https://api.flutter.dev/flutter/material/showLicensePage.html)の中のコードを読んでいくと、[about.dart のこの辺り（_PackageLicensePage）](https://github.com/flutter/flutter/blob/3.19.2/packages/flutter/lib/src/material/about.dart#L770)が参考になりそうだということで、その周辺を参考にしつつ以下のように実装しました。

```dart
import 'package:collection/collection.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'main.g.dart';

void main() => runApp(const ProviderScope(child: MyApp()));

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return const MaterialApp(home: Home());
  }
}

class Home extends StatelessWidget {
  const Home({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ライセンス')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              child: const Text('showLicensePage'),
              onPressed: () => showLicensePage(context: context),
            ),
            const SizedBox(height: 32),
            ElevatedButton(
              child: const Text('カスタムライセンス表示'),
              onPressed: () => Navigator.push(
                  context,
                  MaterialPageRoute(
                      builder: (context) => const CustomLicense())),
            ),
          ],
        ),
      ),
    );
  }
}

class CustomLicense extends ConsumerWidget {
  const CustomLicense({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(title: const Text('カスタムライセンス')),
      body: switch (ref.watch(licenseNotifierProvider)) {
        AsyncData(:final value) => _License(entries: value),
        _ => const Center(child: CircularProgressIndicator()),
      },
    );
  }
}

class _License extends StatelessWidget {
  final List<LicenseEntry> entries;

  const _License({required this.entries});

  @override
  Widget build(BuildContext context) {
    final licenseMap = {};
    for (final entry in entries) {
      for (final package in entry.packages) {
        if (licenseMap.containsKey(package)) {
          licenseMap[package].add(entry);
        } else {
          licenseMap[package] = [entry];
        }
      }
    }

    final licenses = licenseMap.entries
        .map((entry) {
      final List<Widget> list = <Widget>[];

      list.add(
        Text(
          entry.key,
          style: const TextStyle(
            color: Colors.red,
            fontSize: 24,
          ),
        ),
      );

      for (final licenseEntry in entry.value) {
        for (final LicenseParagraph paragraph in licenseEntry.paragraphs) {
          if (paragraph.indent == LicenseParagraph.centeredIndent) {
            list.add(
              Padding(
                padding: const EdgeInsets.only(top: 16.0),
                child: Text(
                  paragraph.text,
                  style: const TextStyle(
                    fontWeight: FontWeight.bold,
                    color: Colors.blue,
                  ),
                  textAlign: TextAlign.center,
                ),
              ),
            );
          } else {
            list.add(
              Padding(
                padding: EdgeInsetsDirectional.only(
                    top: 8.0, start: 16.0 * paragraph.indent),
                child: Text(paragraph.text),
              ),
            );
          }
        }
        list.add(const SizedBox(height: 16));
      }

      list.add(const Divider(height: 15, color: Colors.red));

      return list;
    })
        .flattened
        .toList();

    return ListView.builder(
      itemCount: licenses.length,
      padding: const EdgeInsets.all(16),
      itemBuilder: (_, index) => licenses[index],
    );
  }
}

@riverpod
class LicenseNotifier extends _$LicenseNotifier {
  @override
  Future<List<LicenseEntry>> build() async {
    final entries = <LicenseEntry>[];
    await for (final license in LicenseRegistry.licenses) {
      entries.add(license);
    }
    return entries;
  }
}
```

ここではRiverpodを利用していますが、StatefulWidgetで`LicenseRegistry.licenses.listen((event) { })`で処理しても良いと思います。

パッケージごとにライセンスをズラッと表示したかったため、元の`List<LicenseEntry>`からパッケージ名をkey、`List<LicenseEntry>`というMapを用意し、あとは[PackageLicensePage](https://github.com/flutter/flutter/blob/3.19.2/packages/flutter/lib/src/material/about.dart#L770)と同じような処理をしただけです。
paragraphはインデントの判定ができるようで、Apacheライセンスのように中央表示するようなケースは`if (paragraph.indent == LicenseParagraph.centeredIndent)`のように判定できるようでした。

# 4. おわりに

ライセンスページなんて誰も見ませんよね。(でも実は私はたまに見ています。あ、このアプリこのパッケージ使ってる！！とか)

アプリのユーザは誰もほぼ気にしないところですから、SDK標準のAPIを利用するのが簡単で良いですね。
それでもカスタマイズしたいという場合は、このような方法になりそうです。
