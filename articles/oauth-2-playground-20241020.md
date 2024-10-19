---
title: "【FCM】OAuth 2.0 Playground を使って楽にプッシュ通知をテスト送信したい"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "firebase_messaging"]
published: false
---

# 1. はじめに

FCM (Firebase Cloud Messaging) の動作確認をしたいとき皆さんはどういう方法を取られているでしょうか。最も簡単な方法はFirebaseコンソール (以下) からの送信でしょうか。

![](https://storage.googleapis.com/zenn-user-upload/940faa82f857-20241019.png =300x)

でもこの方法ではペイロードを色々カスタムして確認した場合に対応できませんよね。

他には、公式ドキュメント [認証情報を手動で提供する](https://firebase.google.com/docs/cloud-messaging/auth-server?hl=ja#provide-credentials-manually) に記載されている方法でしょうか。この方法はFirebaseコンソールから「新しい秘密鍵を生成」でダウンロードされるファイルを使ってアクセストークンを入手することができますね。

![](https://storage.googleapis.com/zenn-user-upload/abc8c4498b92-20241019.png =300x)

以下は、アクセストークンを入手する例です。
https://github.com/motucraft/firebase_playground/blob/main/tools/access_token/access_token.js

このように入手したアクセストークンを使って、curlや他のHTTPクライアントツールを使用し、Firebaseのエンドポイントに対してPOSTリクエストを送信して、カスタムペイロードを確認できます。

う〜ん、少し面倒ですね。わざわざアクセストークンを入手するコードを実行しなければなりませんし、セキュリティ上の観点からも秘密鍵を入手するのは可能な限り避けたいですよね。

ということで、Googleの [OAuth 2.0 Playground](https://developers.google.com/oauthplayground/) を使用してもっと簡単にテストメッセージを送信してみたいと思います。OAuth 2.0 Playgroundでは、必要な権限を設定するだけでアクセストークンを取得でき、そのまま簡単にFirebaseのAPIにリクエストを送ることができます。

# 2. プッシュ通知を受信するアプリ

FCMを受信するアプリは、以下のコードを使用します。ほぼ [firebase_messaging](https://pub.dev/packages/firebase_messaging) のexampleです。

https://github.com/motucraft/firebase_playground/blob/main/lib/messaging/main_messaging.dart

![](https://storage.googleapis.com/zenn-user-upload/240da6709162-20241019.gif =300x)

OAuth 2.0 Playground からメッセージを送信し、上記のような挙動を確認します。

# 3. OAuth 2.0 Playground からメッセージを送信する

[OAuth 2.0 Playground](https://developers.google.com/oauthplayground/) へアクセスし、`Firebase Cloud Messaging API v1` の `https://www.googleapis.com/auth/cloud-platform` を選択し、`Authorize APIs`を選択します。

![](https://storage.googleapis.com/zenn-user-upload/21a62d7922b9-20241019.png =300x)

すると、GoogleアカウントへのログインおよびOAuthの権限付与の確認が入ります。

![](https://storage.googleapis.com/zenn-user-upload/04f8e70bdfd7-20241019.png =300x)

許可すると以下のように `Step 2 Exchange authorization code for tokens` の画面へ遷移しますので、`Exchange authorization code for tokens` を選択します。これにより、アクセストークンとリフレッシュトークンが生成されます。

![](https://storage.googleapis.com/zenn-user-upload/3399e230c0a0-20241019.png =300x)

![](https://storage.googleapis.com/zenn-user-upload/f39ef79d6b49-20241019.png =300x)

このように、`Step 3 Configure request to API` の画面へ遷移しました。

ここで、HTTP Method は `POST` を選択し、Request URI は以下を入力します。`<Project Id>` は、Firebaseコンソールから確認することができますので自身の環境のものへ置き換えます。

```shell
https://fcm.googleapis.com/v1/projects/<Project Id>/messages:send
```

次に、`Enter request body` を選択すると以下のような画面が表示されますので、ペイロードのJSONを記載して `Send the request` を選択するとプッシュ通知が送信されます。

![](https://storage.googleapis.com/zenn-user-upload/45f102862699-20241019.png =300x)

ここでは以下のようなペイロードを使用しました。

```json
{
  "message": {
    "token": "<Device Token>",
    "notification": {
      "title": "test1",
      "body": "test message 1."
    },
    "apns": {
      "payload": {
        "aps": {
          "badge": 5,
          "sound": "default"
        }
      }
    },
    "data": {
      "page": "dialog",
      "id": "123"
    }
  }
}
```

`<Device Token>` には、アプリ内で取得したデバイストークンを指定します。今回利用したコードの以下が該当します。

https://github.com/motucraft/firebase_playground/blob/f2e49b670c9c350068d17e570bc723cb089868d6/lib/messaging/main_messaging.dart#L370-L384

`Send the request` で、このように送信成功しプッシュ通知を受信できました。

![](https://storage.googleapis.com/zenn-user-upload/8ff0160b8615-20241019.png =300x)

もしHTTPステータスコードが403で応答される場合は、IAMの権限が付与されていないなどの可能性がありそうです。

# 4. おわりに

今回は、FCMのテスト送信を簡単に行うために、OAuth 2.0 Playground を使った方法を試しました。Firebaseコンソールを使用する以外にも、アクセストークンを取得して、curlやHTTPクライアントツールを使ってペイロードをカスタマイズする方法もありますが、OAuth 2.0 Playground を使うことでより簡単にプッシュ通知のテストを行うことができます。

もしFCMのテストでペイロードを細かく設定したり、複数のメッセージを手軽に送信したい場合にはこの方法が便利かなと思います。
