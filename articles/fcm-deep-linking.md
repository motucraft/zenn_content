---
title: "【Flutter】go_routerとFCMのプッシュ通知でディープリンク"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "deep linking", "go_router", "firebase_messaging", "FCM"]
published: true
---

# 1. はじめに

以下の記事で、go_routerにて宣言的にダイアログやボトムシートを表示しました。

https://zenn.dev/motu2119/articles/go-router-page

ダイアログやボトムシートを宣言的にオープンできるようになると、deep linkに対応できるようになると思いますので、
今回はFCMのプッシュ通知を開いたときにダイアログやボトムシートへディープリンクさせてみたいと思います。

# 2. 挙動

|                                     iOS実機                                      |                                 Androidエミュレータ                                  |
|:------------------------------------------------------------------------------:|:------------------------------------------------------------------------------:|
| ![](https://storage.googleapis.com/zenn-user-upload/5774804fa0ea-20240304.gif) | ![](https://storage.googleapis.com/zenn-user-upload/c7646dd92eb7-20240304.gif) |
|             バックグラウンド通知を処理しています。通知をタップするとボトムシートが開いた状態でアプリが起動しています。              |                    フォアグラウンド通知を処理しています。通知をタップするとダイアログが開きます。                     |

このようにdeep linking対応を行うことで、プッシュ通知をタップしたときに該当するお知らせを開くような挙動を実現できます。

# 3. コード

FCMについては色々なところで解説されていますし、[Firebaseのドキュメント](https://firebase.google.com/docs/cloud-messaging/flutter/client?hl=ja)や[firebase_messagingのexample](https://github.com/firebase/flutterfire/tree/master/packages/firebase_messaging/firebase_messaging/example)を参照すれば、それほど苦労なく実装できると思います。

([flutter_hooks](https://pub.dev/packages/flutter_hooks)を利用していますが、本件には特に関係ありません。Flutter Webの対応は行っていません。)
```dart
import 'dart:convert';
import 'dart:developer';

import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:firebase_playground/firebase_options.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:go_router/go_router.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'main_messaging.g.dart';

/// Initialize the [FlutterLocalNotificationsPlugin] package.
late FlutterLocalNotificationsPlugin _flutterLocalNotificationsPlugin;

/// Create a [AndroidNotificationChannel] for heads up notifications
late AndroidNotificationChannel _channel;

bool _isFlutterLocalNotificationsInitialized = false;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);

  if (!kIsWeb) {
    // iOSおよびAndroidでのローカル通知設定
    await _setupFlutterNotifications();
  }

  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
  final fcmToken = await FirebaseMessaging.instance.getToken();
  log('fcmToken=$fcmToken');

  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp.router(
      routerConfig: ref.watch(routingProvider),
      theme: ThemeData(
        bottomSheetTheme: const BottomSheetThemeData(
          surfaceTintColor: Colors.transparent,
        ),
      ),
    );
  }
}

@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Androidでのローカル通知表示
  _showFlutterNotification(message);
}

Future<void> _setupFlutterNotifications() async {
  if (_isFlutterLocalNotificationsInitialized) {
    return;
  }

  _channel = const AndroidNotificationChannel(
    'default_notification_channel_id',
    'メッセージ通知',
    description: 'ここにチャンネルの説明を記載',
    importance: Importance.high,
  );

  _flutterLocalNotificationsPlugin = FlutterLocalNotificationsPlugin();

  /// Create an Android Notification Channel.
  ///
  /// We use this channel in the `AndroidManifest.xml` file to override the
  /// default FCM channel to enable heads up notifications.
  await _flutterLocalNotificationsPlugin
      .resolvePlatformSpecificImplementation<
      AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(_channel);

  /// Update the iOS foreground notification presentation options to allow
  /// heads up notifications.
  await FirebaseMessaging.instance.setForegroundNotificationPresentationOptions(
    alert: true,
    badge: true,
    sound: true,
  );
  _isFlutterLocalNotificationsInitialized = true;
}

void _showFlutterNotification(RemoteMessage message) {
  RemoteNotification? notification = message.notification;
  AndroidNotification? android = message.notification?.android;
  if (notification != null && android != null && !kIsWeb) {
    _flutterLocalNotificationsPlugin.show(
      notification.hashCode,
      notification.title,
      notification.body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          _channel.id,
          _channel.name,
          channelDescription: _channel.description,
          // TODO Android用アイコン設定
          icon: '@mipmap/ic_launcher',
        ),
      ),
      // FCMのカスタムデータをJSONエンコードしてペイロードへ渡す
      payload: jsonEncode(message.data),
    );
  }
}

class Home extends HookWidget {
  const Home({super.key});

  @override
  Widget build(BuildContext context) {
    final fcmToken = useState<String?>(null);
    useOnAppLifecycleStateChange((_, current) async {
      if (current == AppLifecycleState.resumed) {
        fcmToken.value = await FirebaseMessaging.instance.getToken();
        log('fcmToken=${fcmToken.value}');
        // TODO トークンの保存、バックエンドへの連携など
      }
    });

    useEffect(() {
      // パーミッションリクエスト
      FirebaseMessaging.instance.requestPermission();

      // アプリ終了状態からメッセージをタップして開いた場合
      FirebaseMessaging.instance.getInitialMessage().then((message) {
        log('getInitialMessage');
        _goNamed(context, message?.data, 'getInitialMessage');
      });

      // バックグラウンド状態からメッセージをタップして開いた場合
      // iOSではフォアグラウンド通知のタップでもトリガーされる（Androidのフォアグラウンド通知のタップはトリガーされない）
      FirebaseMessaging.onMessageOpenedApp.listen((message) {
        log('onMessageOpenedApp');
        _goNamed(context, message.data, 'onMessageOpenedApp');
      });

      // フォアグラウンド状態でメッセージを受信した場合のメッセージ表示
      FirebaseMessaging.onMessage.listen(_showFlutterNotification);

      // Androidのフォアグラウンド通知をタップした際にトリガーされる
      _flutterLocalNotificationsPlugin.initialize(
        const InitializationSettings(
          // TODO Android用アイコン設定
          android: AndroidInitializationSettings('@mipmap/ic_launcher'),
          iOS: DarwinInitializationSettings(),
        ),
        onDidReceiveNotificationResponse: (details) {
          log('onDidReceiveNotificationResponse');
          final payload = details.payload;
          if (payload?.isNotEmpty == true) {
            _goNamed(context, jsonDecode(payload!),
                'onDidReceiveNotificationResponse');
          }
        },
      );

      return null;
    }, []);

    return Scaffold(
      appBar: AppBar(
        title: const Text('FCM'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(8.0),
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton(
                child: const Text('open dialog'),
                onPressed: () => context.goNamed(RoutingConfig.dialog.name,
                    pathParameters: {'id': 'dummy'}),
              ),
              const SizedBox(height: 24),
              ElevatedButton(
                child: const Text('open bottom sheet'),
                onPressed: () => context.goNamed(RoutingConfig.bottomSheet.name,
                    pathParameters: {'id': 'dummy'}),
              ),
            ],
          ),
        ),
      ),
    );
  }

  void _goNamed(BuildContext context, Map<String, dynamic>? data, String from) {
    if (!context.mounted || data == null || data.isEmpty) {
      return;
    }

    final page = data['page'];
    final id = data['id'];

    if (context.mounted && page != null && id != null) {
      context.goNamed(page, pathParameters: {'id': '$from $id'});
    }
  }
}

@riverpod
GoRouter routing(RoutingRef ref) {
  final router = GoRouter(
    routes: [
      GoRoute(
        path: RoutingConfig.home.path,
        name: RoutingConfig.home.name,
        pageBuilder: (_, __) => const MaterialPage(child: Home()),
        routes: [
          GoRoute(
            path: RoutingConfig.dialog.path,
            name: RoutingConfig.dialog.name,
            pageBuilder: (_, state) => DialogPage(
              builder: (_) {
                final id = state.pathParameters['id'];
                if (id == null || id.isEmpty) {
                  throw UnsupportedError('ID does not exist.');
                }

                return SampleContent(id: id);
              },
            ),
          ),
          GoRoute(
            path: RoutingConfig.bottomSheet.path,
            name: RoutingConfig.bottomSheet.name,
            pageBuilder: (_, state) => BottomSheetPage(
              builder: (_) {
                final id = state.pathParameters['id'];
                if (id == null || id.isEmpty) {
                  throw UnsupportedError('ID does not exist.');
                }

                return SampleContent(id: id);
              },
            ),
          ),
        ],
      ),
    ],
  );

  ref.onDispose(router.dispose);
  return router;
}

enum RoutingConfig {
  home('/', 'home'),
  dialog('dialog/:id', 'dialog'),
  bottomSheet('bottom-sheet/:id', 'bottomSheet');

  final String path;
  final String name;

  const RoutingConfig(this.path, this.name);
}

class SampleContent extends StatelessWidget {
  final String id;

  const SampleContent({super.key, required this.id});

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      backgroundColor: Colors.white,
      surfaceTintColor: Colors.transparent,
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text('id=$id'),
          const SizedBox(height: 12),
          ElevatedButton(
            onPressed: context.pop,
            child: const Text('close'),
          ),
        ],
      ),
    );
  }
}

class DialogPage<T> extends Page<T> {
  final Offset? anchorPoint;
  final Color? barrierColor;
  final bool barrierDismissible;
  final String? barrierLabel;
  final bool useSafeArea;
  final CapturedThemes? themes;
  final WidgetBuilder builder;

  const DialogPage({
    required this.builder,
    this.anchorPoint,
    this.barrierColor = Colors.black54,
    this.barrierDismissible = true,
    this.barrierLabel,
    this.useSafeArea = true,
    this.themes,
    super.key,
    super.name,
    super.arguments,
    super.restorationId,
  });

  @override
  Route<T> createRoute(BuildContext context) {
    return DialogRoute<T>(
      context: context,
      settings: this,
      builder: builder,
      anchorPoint: anchorPoint,
      barrierColor: barrierColor,
      barrierDismissible: barrierDismissible,
      barrierLabel: barrierLabel,
      useSafeArea: useSafeArea,
      themes: themes,
    );
  }
}

class BottomSheetPage<T> extends Page<T> {
  final WidgetBuilder builder;
  final Offset? anchorPoint;
  final String? barrierLabel;
  final CapturedThemes? themes;

  const BottomSheetPage({
    required this.builder,
    this.anchorPoint,
    this.barrierLabel,
    this.themes,
  });

  @override
  Route<T> createRoute(BuildContext context) {
    return ModalBottomSheetRoute(
      settings: this,
      builder: builder,
      anchorPoint: anchorPoint,
      barrierLabel: barrierLabel,
      isScrollControlled: true,
      backgroundColor: Colors.white,
      constraints: BoxConstraints(
        maxHeight: MediaQuery.sizeOf(context).height / 2,
      ),
      useSafeArea: true,
      showDragHandle: true,
      elevation: 1.0,
    );
  }
}
```

画面遷移は以下の箇所で実行しています。FCMのカスタムデータを使用してディープリンクさせています。

```dart
  void _goNamed(BuildContext context, Map<String, dynamic>? data, String from) {
    if (!context.mounted || data == null || data.isEmpty) {
      return;
    }

    final page = data['page'];
    final id = data['id'];

    if (context.mounted && page != null && id != null) {
      context.goNamed(page, pathParameters: {'id': '$from $id'});
    }
  }
```

# 4. FCMメッセージ送信

Firebaseコンソールから簡単に送信できますので、以下のようにカスタムデータを設定してテストメッセージを送信しました。
テストメッセージの送信で必要となるトークンは、`FirebaseMessaging.instance.getToken()`で取得してコンソールへログ出力していますのでそれを使用しました。

![](https://storage.googleapis.com/zenn-user-upload/b6a9f116e168-20240304.png)

# 5. おわりに

go_routerのおかげで、お手軽にディープリンクを実装できそうですね。
ただし、ここではただ動かしたというだけで異常系など何も考えていませんので、それなりにハマりどころはありそうです。
