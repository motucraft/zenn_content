---
title: "【Flutter】Firebase Data Connectを試す"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "firebase_data_connect"]
published: false
---

# 1. はじめに

もう半年ほど前になってしまいますが、[Google I/O 2024にてFirebase Data Connectが発表](https://firebase.blog/posts/2024/05/whats-new-at-google-io)されました。
Firebase Data Connectを使用すると、Cloud SQLでホストされているPostgresSQLへアプリから直接接続できるようになるということですね。
FirestoreではなくRDBを使いたいという要望がかなり高かったのでしょうね。Supabaseへの対抗ということもあるのでしょうか。

触ってみたいと思いつつなかなか時間が取れずにいましたが、1ヶ月ほど前に[firebase_data_connect](https://pub.dev/packages/firebase_data_connect)が公開されたこともあって今回お試ししてみることにしました。

Firebase Data ConnectはまだGAされておらず(2024年10月現在)、限定公開プレビュー版というステータスです。
そのためEarly Accessの申し込みが必要になります。ここではEarly Accessの申請が通っていること、Firebaseのプロジェクト作成やFlutterのプロジェクトの接続が終わっていることを前提に試していこうと思います。

# 2. ゴール

Data Connectがサンプルとして用意しているスキーマ(Movie, MovieMetadata)を使って、以下のように映画の一覧を表示(query)します。
また、画面右上の+アイコンにて表示されるダイアログからmutationして映画の情報を追加できるようにします。

| mutation | query |
|:--------:|:-----:|
| ![](https://storage.googleapis.com/zenn-user-upload/670e1b421a42-20241013.gif) | ![](https://storage.googleapis.com/zenn-user-upload/e535f3fd1338-20241013.gif) |

コードはこちらです。

https://github.com/motucraft/firebase_playground/blob/main/lib/dataconnect/main_dataconnect.dart

パッケージは、firebase_data_connectを利用しています。

https://pub.dev/packages/firebase_data_connect

