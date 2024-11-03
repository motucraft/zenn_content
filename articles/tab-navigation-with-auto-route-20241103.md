---
title: "【Flutter】auto_route で Tab Navigation (BottomNavigationBar)"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "auto_route"]
published: true
---

# 1. はじめに

[auto_route](https://pub.dev/packages/auto_route) を利用して [Tab Navigation](https://pub.dev/packages/auto_route#tab-navigation) を行います。中でも今回はアプリでよく利用する [BottomNavigationBar](https://api.flutter.dev/flutter/material/BottomNavigationBar-class.html) 周辺を探ってみようと思います。
私の場合は、これまで [go_router](https://pub.dev/packages/go_router) を使用することが多かったのですが、それは"First Partyだから"という程度の理由しかありませんでしたので、auto_route も味見してみようという趣旨です。

# 2. ゴール

今回のコードはこちらに置きました。ボトムナビを表示するだけだと味気ないので、画面の状態維持なども試してみました。

https://github.com/motucraft/auto_route_playground/tree/main/lib/tab_navigation

auto_route のバージョンはこちらを使用しました。（2024年11月時点の最新）
```yaml
dependencies:
  auto_route: ^9.2.2
dev_dependencies:
  auto_route_generator: ^9.0.0
```

## 2.1 BottomNavigationBar を用意する

一般的なアプリにとって定番だと思いますので、このような BottomNavigationBar を用意してみます。

![](https://storage.googleapis.com/zenn-user-upload/d73c3a056471-20241102.gif =300x)

## 2.2 宣言的にダイアログやボトムシートを表示する

こちらの内容とほぼ同じです。
https://zenn.dev/motu2119/articles/go-router-page

各画面内に表示するか、BottomNavigationBar も含めてその上に表示するかという違いがあります。

![](https://storage.googleapis.com/zenn-user-upload/b6fd46666644-20241102.gif =300x)

## 2.3 BottomNavigationBar の状態を維持する

ボトムナビでタブを切り替えても、移動前の状態を保持しておきたい場合があります。このように、各タブの画面状態を維持します。

![](https://storage.googleapis.com/zenn-user-upload/b05703812df4-20241102.gif =300x)

そして、Home画面への移動を契機に他画面を初期状態へ戻します。
（各画面の状態が残ったままで良ければ不要です）

![](https://storage.googleapis.com/zenn-user-upload/c977faa0e697-20241102.gif =300x)

ダイアログやボトムシートについても、タブ内の表示であれば表示したまま他のタブへ移動することもできます。

![](https://storage.googleapis.com/zenn-user-upload/e791ce50688a-20241103.gif =300x)

## 2.4 BottomNavigationBar 表示/非表示を制御する

画面によっては、BottomNavigationBar を表示せずに全画面表示としたい画面もあると思います。そこで、画面ごとにボトムナビの表示/非表示を制御します。

![](https://storage.googleapis.com/zenn-user-upload/17b0f3fe78e7-20241103.gif =300x)

## 2.5 BottomNavigationBar のタブ間で遷移する

ボトムナビのタブ間での遷移を行います。サブルートから他のタブのルートページへ、またはサブルートから他のタブのサブルートへ遷移する動作です。

| 1. サブルートから他のタブのルートページへ                                                               |                                2. サブルートから他のタブのサブルートへ                                 |
|:-------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------:|
| ![](https://storage.googleapis.com/zenn-user-upload/cac4f97e5c12-20241102.gif =300x) | ![](https://storage.googleapis.com/zenn-user-upload/7cf2344b06fc-20241102.gif =300x) |

1. FavoritesとPosts間を移動しています
2. FavoritesとPosts間でお互いのサブルートからサブルートへ移動しています

## 2.6 タブ再タップでナビゲーションスタックをルートへ戻す

サブルートにいる状態でボトムナビのアイコンをタップすると、そのタブのルート画面に戻ります。

![](https://storage.googleapis.com/zenn-user-upload/6e661b7a6521-20241103.gif =300x)

# 3. コードスニペット

## 3.1 BottomNavigationBar を用意する

ルーティングの定義はこちらです。
https://github.com/motucraft/auto_route_playground/blob/main/lib/tab_navigation/router/router.dart

`auto_route`では、ルート間でネストされたナビゲーションを定義できます。ドキュメントにある [Nested Navigation](https://pub.dev/packages/auto_route#nested-navigation) の例に基づき、`BaseRoute`の`children`として各タブのルートを定義しています。これにより、タブごとに異なるナビゲーションスタックを持たせることが可能です。タブごとに異なるスタックを持たせることで、各タブで独立したナビゲーションを実現できます。

`BaseRoute (BaseScreen)`はこちらです。ここで BottomNavigationBar を構築しています。
https://github.com/motucraft/auto_route_playground/blob/main/lib/tab_navigation/screens/base/base_screen.dart

[BottomNavigationBar](https://api.flutter.dev/flutter/material/BottomNavigationBar-class.html) は、

> The bottom navigation bar's type changes how its items are displayed. If not specified, then it's automatically set to BottomNavigationBarType.fixed when there are less than four items, and BottomNavigationBarType.shifting otherwise.

ということですので、`BottomNavigationBarType.fixed` を指定しています。
また、タブの状態維持を行いたいため、[AutoTabsRouter.builder](https://pub.dev/documentation/auto_route/latest/auto_route/AutoTabsRouter/AutoTabsRouter.builder.html) を利用しています。

## 3.2 宣言的にダイアログやボトムシートを表示する

上でリンクした `router.dart` の `_dialogCustomRoute`、`_bottomSheetCustomRoute`にて、`CustomRoute` を生成しています。これにより、特定のタブ内で必要に応じてダイアログやボトムシートを呼び出しやすくなり、宣言的なルート管理が可能です。

特定のタブでのみ表示するダイアログやボトムシートは、各タブ内で宣言的にルートを設定することで実現しています。例えば、`HomeShellRoute`には`SampleDialogRoute`と`SampleBottomSheetRoute`が含まれており、Homeタブ内でこれらを呼び出せます。
一方、`GlobalSampleDialogRoute`や`GlobalSampleBottomSheetRoute`はグローバルに設定されているため、どのタブでも共通のダイアログやボトムシートとして利用可能です。

## 3.3 BottomNavigationBar の状態を維持する

AutoTabsRouter.builder の builder コールバック内で、[IndexedStack](https://api.flutter.dev/flutter/widgets/IndexedStack-class.html) を使用しています。
https://github.com/motucraft/auto_route_playground/blob/49001df16b73487bc54c1d78015fff89def0b8af/lib/tab_navigation/screens/base/base_screen.dart#L11-L26

IndexedStack は、非アクティブなタブのウィジェットツリーを保持し、アクティブにしたときにその状態を復元するために使用しています。IndexedStack を使うことで、再描画が発生しないためタブを切り替えても状態が維持されます。
ここで、「2. ゴール」に記載したとおり、

> そして、Home画面への移動を契機に他画面を初期状態へ戻します。

を実現するために、Home画面を表示したとき（tabsRouter.activeIndex == 0 のとき）は IndexedStack を使用しないようにしています。

## 3.4 BottomNavigationBar 表示/非表示を制御する

`auto_route` では、ルートごとに `meta` 属性を使って静的データを渡すことができます。`meta` は、特定のルートに対してカスタマイズ情報を持たせるような使い方ができます。ここでは、`BottomNavigationBar` の表示/非表示の制御で使用します。
`auto_route` パッケージ作者のこちらのコメントが参考になりました。

https://github.com/Milad-Akarie/auto_route_library/issues/981#issuecomment-1052004915

以下のルートでは BottomNavigationBar を表示しない（全画面表示する）ために、`meta: {hideBottomNavKey: true}` を設定しています。
https://github.com/motucraft/auto_route_playground/blob/49001df16b73487bc54c1d78015fff89def0b8af/lib/tab_navigation/router/router.dart#L68-L71

そして、ここで `meta` を参照して BottomNavigationBar の表示/非表示を制御しています。
https://github.com/motucraft/auto_route_playground/blob/49001df16b73487bc54c1d78015fff89def0b8af/lib/tab_navigation/screens/base/base_screen.dart#L29-L33

## 3.5 BottomNavigationBar のタブ間で遷移する

ここで、Favorites タブのサブルートから、Posts タブへの遷移を行っています。サブルートへ遷移させる場合は、`children` を指定します。
https://github.com/motucraft/auto_route_playground/blob/49001df16b73487bc54c1d78015fff89def0b8af/lib/tab_navigation/screens/favorites/favorites_screen.dart#L60-L74

同様に、Posts タブのサブルートから、Favorites タブへの遷移を行っています。
https://github.com/motucraft/auto_route_playground/blob/49001df16b73487bc54c1d78015fff89def0b8af/lib/tab_navigation/screens/posts/posts_screen.dart#L60-L74

[navigate](https://pub.dev/documentation/auto_route/latest/auto_route/RoutingController/navigate.html) メソッドは、指定したルートが既にスタックにある場合はそのルートまでポップし、スタックにない場合は新規にルートを追加します。

## 3.6 タブ再タップでナビゲーションスタックをルートへ戻す

コードはこちらが該当します。
https://github.com/motucraft/auto_route_playground/blob/49001df16b73487bc54c1d78015fff89def0b8af/lib/tab_navigation/screens/base/base_screen.dart#L36-L45

タブが表示されている状態で同じ BottomNavigationBar アイコンを再タップすると、そのタブのナビゲーションスタックをルートまで戻す動作を実装しています。[popUntilRoot](https://pub.dev/documentation/auto_route/latest/auto_route/StackRouter/popUntilRoot.html) にて、同じタブがタップされた際にスタックがクリアされてナビゲーションスタックのルートへ戻ることになります。

また、異なるタブがタップされた場合には、[setActiveIndex](https://pub.dev/documentation/auto_route/latest/auto_route/TabsRouter/setActiveIndex.html) によって、そのタブがアクティブ化されます。

# 4. おわりに

auto_route を使った BottomNavigationBar でのタブナビゲーション周りを探ってみました。
タブの状態を維持する方法やルートごとのmeta情報を活用したナビゲーションの表示制御、同じタブを再タップした際のスタックのリセットといったこともできたため、かゆいところにも手が届きそうな感触を得ました。

他には、ルートガードやディープリンク周りも試してみたいところです。

# 5. 参考

- [【Flutter】ボトムナビゲーションバーを表示したまま画面遷移したい【auto_route】](https://zenn.dev/susatthi/articles/20230427-095829-flutter-auto-route)
  [すささん](https://zenn.dev/susatthi) の記事を参考にさせていただきました🙇‍♂️
