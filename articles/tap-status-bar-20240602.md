---
title: "【Flutter】ShellRoute/StatefulShellRouteを使用中にiOSのステータスバータップを検出する"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "go_router"]
published: true
---

# 1. はじめに

iOSにおいて、ステータスバーをタップしたときにスクロールをトップへ戻す動きがあると思います。iOSユーザは、頻繁にステータスバーをタップしてScroll To Topさせることを行うと思いますので、UXとしても重要な点なのかなと思っています。

例えば単純な以下のコードは、実装上特に何も考慮せずともiOSのステータスバーをタップするとスクロールがトップへ戻る挙動になります。

https://github.com/motucraft/tap_status_bar/blob/main/lib/ok_without_go_router.dart

これはScaffoldが[PrimaryScrollController](https://api.flutter.dev/flutter/widgets/PrimaryScrollController-class.html)を使用して機能させていると考えれば良さそうです。

> Inheriting this ScrollController can provide default behavior for scroll views in a subtree. For example, the Scaffold uses this mechanism to implement the scroll-to-top gesture on iOS.

では、以下のように[go_router](https://pub.dev/packages/go_router)の[ShellRoute](https://pub.dev/documentation/go_router/latest/go_router/ShellRoute-class.html)や[StatefulShellRoute](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellRoute-class.html)を利用した場合はどうでしょうか。

- ShellRouteを使用したサンプル

https://github.com/motucraft/tap_status_bar/blob/main/lib/ng_with_shell_route.dart

- StatefulShellRouteを使用したサンプル

https://github.com/motucraft/tap_status_bar/blob/main/lib/ng_with_stateful_shell_route.dart

上記コードのようにShellRoute/StatefulShellRouteをすると、ステータスバーをタップしてもスクロールが動きません。
そこで、issueを起票しました。ステータスバーのタップを検出する方法があるのでしょうか？

https://github.com/flutter/flutter/issues/149484

（おそらく既存のissueも存在しそうで重複だと言われてクローズされるかもしれませんが）

# 2. 正しいのかどうか自信の無いソリューション

PrimaryScrollControllerは、以下のように[InheritedWidget](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)であり、
`Creates a widget that associates a [ScrollController] with a subtree.`ということのようです。

```dart
class PrimaryScrollController extends InheritedWidget {
  /// Creates a widget that associates a [ScrollController] with a subtree.
  const PrimaryScrollController({
```

それならば、[ScrollControllerのattachメソッド](https://api.flutter.dev/flutter/widgets/ScrollController/attach.html)を使ってなんとかできないものかと模索しました。

`attach`メソッドの引数には[ScrollPosition](https://api.flutter.dev/flutter/widgets/ScrollPosition-class.html)が必要です。
[ScrollPositionWithSingleContext](https://api.flutter.dev/flutter/widgets/ScrollPositionWithSingleContext-class.html)を継承したダミーのScrollPositionを渡しておけば、ステータスバータップ時に[animateTo](https://api.flutter.dev/flutter/widgets/ScrollPosition/animateTo.html)が呼び出されるのではないのかと。
`animateTo`が呼び出されてさえくれれば、あとはそこをコールバックで処理してなんとかなりそうだ、と考えて模索したのが下記のコードです。

- 作成したTapStatusBarNotifierクラス

https://github.com/motucraft/tap_status_bar/blob/main/lib/tap_status_bar_notifier/tap_status_bar_notifier.dart

以下は、TapStatusBarNotifierを使用したコードです。`onTapStatusBar`コールバック内で自前でスクロールトップさせています。
 
- ShellRoute

https://github.com/motucraft/tap_status_bar/blob/main/lib/tap_status_bar_notifier/ok_shell_route.dart

- StatefulShellRoute

https://github.com/motucraft/tap_status_bar/blob/main/lib/tap_status_bar_notifier/ok_stateful_shell_route.dart

`matchedLocation`を確認して、現在表示中の画面であればスクロールをトップに戻しています。この確認を忘れてしまうと、StatefulShellRouteの場合は他の画面も一緒にスクロールトップされてしまうことになります。（逆に言えば、そんなこともできるのか！）

```dart
onTapStatusBar: () {
  if (GoRouter.of(context).routerDelegate.currentConfiguration.last.matchedLocation == '/home') {
    _controller.animateTo(0, duration: const Duration(milliseconds: 300), curve: Curves.linear);
  }
},
```

以下は、StatefulShellRouteの挙動です。Like画面のステータスバータップにてスクロールトップしても、Home画面には影響を与えていないことを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/5f7b1e39a027-20240602.gif =300x)

# 3. おわりに

一応やりたいことはできたのですが、これが正しい対応なのか最善なのか自信が持てません。issueへのコメントを待ちたいと思います。どなたかよろしくお願い致します🙇‍♂

https://github.com/flutter/flutter/issues/149484
