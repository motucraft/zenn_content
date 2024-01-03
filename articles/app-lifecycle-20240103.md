---
title: "【Flutter】Riverpodを利用してAppLifecycleStateとconnectivity_plusのネットワーク接続を検出する"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "AppLifecycle", "connectivity_plus"]
published: false
---

# 1. はじめに

Flutterにはアプリケーションの実行状態を示す[AppLifecycleState](https://api.flutter.dev/flutter/dart-ui/AppLifecycleState.html)があります。
Riverpodを利用しつつ、この`AppLifecycleState`を入手したいと考えました。

公式ドキュメントを見ると、以下のような説明があります。

```
The current application state can be obtained from SchedulerBinding.instance.lifecycleState, and changes to the state can be observed by creating an AppLifecycleListener, or by using a WidgetsBindingObserver by overriding the WidgetsBindingObserver.didChangeAppLifecycleState method.
```

あとは、こちらのRemiさんのtweetを参考にします。
https://x.com/remi_rousselet/status/1486675682491092997?s=20

ついでに、[connectivity_plus](https://pub.dev/packages/connectivity_plus)を利用してネットワーク接続も検出するようにします。

# 2. pubspec.yaml

Riverpodはv3を利用しました。

```yaml
name: app_lifecycle
description: "A new Flutter project."
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.2.3 <4.0.0'

dependencies:
  flutter:
    sdk: flutter

  cupertino_icons: ^1.0.2
  hooks_riverpod: ^3.0.0-dev.3
  flutter_hooks: ^0.20.4
  riverpod_annotation: ^3.0.0-dev.3
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1
  connectivity_plus: ^5.0.2

dev_dependencies:
  flutter_test:
    sdk: flutter

  flutter_lints: ^2.0.0
  riverpod_generator: ^3.0.0-dev.11
  build_runner: ^2.4.7
  custom_lint: ^0.5.7
  riverpod_lint: ^3.0.0-dev.4
  freezed: ^2.4.6
  json_serializable: ^6.7.1

flutter:
  uses-material-design: true
```

# 3. AppLifecycleStateとネットワーク接続を検出するProvider

以下のように実装しました。Widgetからは、`appLifecycleNotifierProvider`をwatchします。

```dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:flutter/material.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'app_lifecycle_notifier.freezed.dart';
part 'app_lifecycle_notifier.g.dart';

@riverpod
FutureOr<AppLifecycle?> appLifecycleNotifier(AppLifecycleNotifierRef ref) {
  final connectivity = ref.watch(connectivityProvider).valueOrNull;
  if (connectivity == null) {
    ref.state = const AsyncLoading<AppLifecycle?>().copyWithPrevious(ref.state);
    return Future.value(ref.state.valueOrNull);
  }

  final appLifecycle = ref.watch(appLifecycleProvider);
  return AppLifecycle(appLifecycleState: appLifecycle, connectivityResult: connectivity);
}

@riverpod
AppLifecycleState appLifecycle(AppLifecycleRef ref) {
  final appLifecycleObserver =
      _AppLifecycleObserver((appLifecycleState) => ref.state = appLifecycleState);

  final widgetsBinding = WidgetsBinding.instance..addObserver(appLifecycleObserver);
  ref.onDispose(() => widgetsBinding.removeObserver(appLifecycleObserver));

  return AppLifecycleState.resumed;
}

class _AppLifecycleObserver extends WidgetsBindingObserver {
  final ValueChanged<AppLifecycleState> _didChangeAppLifecycle;

  _AppLifecycleObserver(this._didChangeAppLifecycle);

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    _didChangeAppLifecycle(state);
    super.didChangeAppLifecycleState(state);
  }
}

@riverpod
Stream<ConnectivityResult> connectivity(ConnectivityRef ref) =>
    Connectivity().onConnectivityChanged;

@freezed
class AppLifecycle with _$AppLifecycle {
  const factory AppLifecycle({
    required AppLifecycleState appLifecycleState,
    required ConnectivityResult connectivityResult,
  }) = _AppLifecycle;
}
```

`appLifecycleProvider`と`_AppLifecycleObserver`は、Remiさんのtweetを参考にしています。
`_AppLifecycleObserver`へコールバックを渡しています。`didChangeAppLifecycleState`に`AppLifecycleState`が渡されてきますので、そこでコールバックを呼び出して`appLifecycleProvider`のstateを更新しています。

あとは、Widgetから`ref.watch(appLifecycleNotifierProvider)`でfreezedのインスタンス`AppLifecycle`を入手できます。
以下はmain.dartのサンプルです。アプリを表示した状態で機内モードにしたり、アプリをバックグラウンドに移行したり、スコープが当たっていない状態（inactive）にしたり諸々確認してみましたが機能しているようでした。

```dart
import 'package:app_lifecycle/app_lifecycle_notifier.dart';
import 'package:flutter/material.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';

void main() {
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const AppLifecycle(),
    );
  }
}

class AppLifecycle extends ConsumerWidget {
  const AppLifecycle({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(),
      body: Center(
        child: switch (ref.watch(appLifecycleNotifierProvider)) {
          AsyncData(:final value) => value == null
              ? const SizedBox()
              : Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text('appLifecycleState=${value.appLifecycleState}'),
                    Text('connectivityResult=${value.connectivityResult}'),
                  ],
                ),
          AsyncError(:final error, :final stackTrace) =>
            Text('error=$error, stackTrace=$stackTrace'),
          _ => const CircularProgressIndicator(),
        },
      ),
    );
  }
}
```

# 4. おわりに

渡しの場合、`AppLifecycleState`を入手したいというモチベーションはFirestoreでした。
Firestoreのsnapshotを利用しているときに、アプリがバックグラウンドへ移行したならsnapshotをキャンセルしたかったためです。

この実装で`AppLifecycleState`（と、ついでにネットワーク接続も）検出できるようになりましたので、Firestore周りに適用していこうと思います。
