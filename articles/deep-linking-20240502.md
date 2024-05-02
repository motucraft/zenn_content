---
title: "【Flutter】ディープリンクでダイアログとボトムシートを開く(Deep linking with go_router)"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "deep linking", "go_router"]
published: false
---

# 1. はじめに

https://zenn.dev/motu2119/articles/go-router-page

以前、上記の記事にて宣言的にダイアログやボトムシートを開くということを行いました。そして、

> ダイアログやボトムシートを宣言的にオープンできるようになると、deep linkに対応できるようになると思います。

記事の中でこのように書いていましたので、今回はdeeplinkを試そうと思います。

カスタムスキームは使用しません。Androidで言うapp link、iOSで言うuniversal linkです。
3.参考のドキュメントにも以下のような記載があります。
- `A app link is a type of deep link that uses http or https and is exclusive to Android devices.`
- `A universal link is a type of deep link that uses http or https and is exclusive to Apple devices.`

# 2. 挙動

以下のような挙動になります。iOSは実機、Androidはエミュレータにて確認しました。
それぞれメモ帳アプリへdeeplinkのURLを記載し、そのURLをタップすることでアプリを起動させています。

|                              iOS (Physical Device)<br/>iPhone 15 Pro                               |                                Android (Emulator)<br/>Pixel8 Pro API 34                                |
|:--------------------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------------------------:|
| ![ios](https://github.com/motucraft/deeplink/assets/35750184/eeaa0803-7e90-4008-a5b8-c1e24f7a4d61) | ![android](https://github.com/motucraft/deeplink/assets/35750184/00b5d889-c97a-4019-97a8-8aaf318f603a) |

今回は他アプリ（メモ帳アプリ）からのdeeplinkの挙動を確認しました。URLをWebへ埋め込むなどすれば、ブラウザからアプリの特定ページへリンクすることもできます。

# 3. 参考

Flutter公式ドキュメントを参考にしていきます。

- [Deep linking](https://docs.flutter.dev/ui/navigation/deep-linking)
- [Set up app links for Android](https://docs.flutter.dev/cookbook/navigation/set-up-app-links)
- [Set up universal links for iOS](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links)

# 4. コード

Flutterのコードとしては、[【Flutter】go_routerで宣言的にダイアログやボトムシートをオープンする](https://zenn.dev/motu2119/articles/go-router-page)に記載したコードと全く同じです。使用したパッケージは、[go_router](https://pub.dev/packages/go_router)のみです。コードはGitHubにも置いています。

https://github.com/motucraft/deeplink

# 5. Android Settings

[Set up app links for Android](https://docs.flutter.dev/cookbook/navigation/set-up-app-links)に従って設定していきます。
既に実装はできていますので、[Customize a Flutter application](https://docs.flutter.dev/cookbook/navigation/set-up-app-links#1-customize-a-flutter-application)はスキップします。

## 5.1 [Modify AndroidManifest.xml](https://docs.flutter.dev/cookbook/navigation/set-up-app-links#2-modify-androidmanifest-xml)

`android/app/src/main/AndroidManifest.xml`を編集します。`<activity>`エレメント内に以下のように追記します。
ここでドメインを記載することになります。私の場合は、`deeplink.motucraft.jp`としました。

```xml
        <activity>
    
            〜省略〜
    
            <meta-data android:name="flutter_deeplinking_enabled" android:value="true" />
            <intent-filter android:autoVerify="true">
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <data android:scheme="http" android:host="deeplink.motucraft.jp" />
                <data android:scheme="https" />
            </intent-filter>
        </activity>
```

## 5.2 [Hosting assetlinks.json file](https://docs.flutter.dev/cookbook/navigation/set-up-app-links#3-hosting-assetlinks-json-file)

以下のように、`assetlinks.json`ファイルを用意しました。`package_name`は、私の場合は`jp.motucraft.deeplink`です。
フィンガープリントは、実際はGoogle Play Consoleから生成することになると思います。今回はお試しですし私だけが実行できれば良いので、ローカルで生成(`./gradlew :app:signingReport`)しました。

```json
[
    {
        "relation": [
            "delegate_permission/common.handle_all_urls"
        ],
        "target": {
            "namespace": "android_app",
            "package_name": "jp.motucraft.deeplink",
            "sha256_cert_fingerprints": [
                "C8:CF:9C:DE:91:80:69:28:5F:DB:AD:2B:78:B8:FA:A9:4C:A1:92:E4:3E:5A:F2:5D:01:61:E1:F3:C5:C0:8E:76"
            ]
        }
    }
]
```

このファイルをサーバへ配置します。ドキュメントに`Host the file at a URL that resembles the following: <webdomain>/.well-known/assetlinks.json`とありますので、それに従います。

そのうち削除するかもしれませんが、以下のとおりです。

http://deeplink.motucraft.jp/.well-known/assetlinks.json

## 5.3 [Testing](https://docs.flutter.dev/cookbook/navigation/set-up-app-links#testing)

adbコマンドにて、以下のようにテスト可能です。

```shell
adb shell 'am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "http://deeplink.motucraft.jp/dialog"' jp.motucraft.deeplink
```

ただし、この方法はドキュメントに記載のあるとおり、`This doesn't test whether the web files are hosted correctly, the command launches the app even if web files are not presented.`ということで、ドメインや`assetlinks.json`ファイルの検証は行われませんので注意が必要です。
`assetlinks.json`ファイルが存在しなくても機能しますので、アプリのルーティングが正しく動作するか否かの確認として使用できると思います。

Android用の設定は以上です。

# 6. iOS Settings

[Set up universal links for iOS](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links)に従って設定していきます。
既に実装はできていますので、[Customize a Flutter application](https://docs.flutter.dev/cookbook/navigation/set-up-app-links#1-customize-a-flutter-application)はスキップします。

## 6.1 [Adjust iOS build settings](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links#2-adjust-ios-build-settings)

ドキュメントに記載されている通り、XCodeから以下のように設定しました。

![](https://storage.googleapis.com/zenn-user-upload/3186f5cc7ed9-20240502.png)

![](https://storage.googleapis.com/zenn-user-upload/a4a78e4e398f-20240502.png)

## 6.2 [Hosting apple-app-site-association file](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links#3-hosting-apple-app-site-association-file)

以下のように、`apple-app-site-association`ファイルを用意しました。`appID`は、`Apple formats the app ID as <team id>.<bundle id>`とのことです。

```shell
{
  "applinks": {
      "apps": [],
      "details": [
      {
        "appID": "PNRW3X92CC.jp.motucraft.deeplink",
        "paths": ["*"]
      }
    ]
  }
}
```

`Host the file at a URL that resembles the following: <webdomain>/.well-known/apple-app-site-association`ということですので、Androidの場合と同様にサーバへ配置します。

こちらもそのうち削除するかもしれませんが、以下のとおりです。

http://deeplink.motucraft.jp/.well-known/apple-app-site-association

ファイルを配置しても即時有効となるわけではなく、以下ドキュメント記載のとおりですので注意が必要です。
> It might take up to 24 hours before Apple's Content Delivery Network (CDN) requests the apple-app-site-association (AASA) file from your web domain. The universal link won't work until the CDN requests the file. To bypass Apple's CDN, check out the alternate mode section.


## 6.3 [Testing](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links#testing)

シミュレータにて、以下のようにテスト可能です。こちらは、Androidのadbコマンドの場合とは異なり、apple-app-site-associationファイルの配置(およびAppleのCDNのリクエスト完了)が必要です。

```shell
xcrun simctl openurl booted https://deeplink.motucraft.jp/dialog
```

iOS用の設定は以上です。

# 7. おわりに

Flutter公式ドキュメントのとおりに対応していっただけですが、特に迷うこともなく[go_router](https://pub.dev/packages/go_router)のみでdeeplinkの挙動を確認できました。
宣言的にダイアログやボトムシートをオープンするようにしておくことで、deeplinkに対応しやすくなります。
