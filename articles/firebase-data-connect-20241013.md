---
title: "【Flutter】Firebase Data Connect を試す"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "firebase_data_connect"]
published: true
---

# 1. はじめに

時間が経つのは早いもので...もう半年ほど前になってしまいますが、[Google I/O 2024にてFirebase Data Connectが発表](https://firebase.blog/posts/2024/05/whats-new-at-google-io)されました。
Firebase Data Connectを使用すると、Cloud SQLでホストされているPostgresSQLへアプリから直接接続できるようになるということですね。
これは、RDBを使いたいという要望がかなり高かったのでしょうね。加えて、Supabaseへの対抗ということもあるのでしょうか。

触ってみたいと思いつつなかなか時間が取れずにいましたが、1ヶ月ほど前に[firebase_data_connect](https://pub.dev/packages/firebase_data_connect)が公開されたこともあって今回お試ししてみることにしました。

Firebase Data ConnectはまだGAされておらず(2024年10月現在)、限定公開プレビュー版というステータスです。
そのためEarly Accessの申し込みが必要になります。ここではEarly Accessの申請が通っていること、Firebaseのプロジェクト作成やFlutterのプロジェクトの接続が終わっていることを前提に試していこうと思います。

# 2. ゴール

Data Connectがサンプルとして用意しているスキーマ(Movie, MovieMetadata)を使って、以下のように映画の一覧をqueryします。また、画面右上の + アイコンにて表示されるダイアログからmutationを実行して映画の情報を追加できるようにします。

| mutation | query |
|:--------:|:-----:|
| ![](https://storage.googleapis.com/zenn-user-upload/670e1b421a42-20241013.gif) | ![](https://storage.googleapis.com/zenn-user-upload/e535f3fd1338-20241013.gif) |

コードはこちらです。お試しですので、エラー処理などは入っていません。

https://github.com/motucraft/firebase_playground/blob/main/lib/dataconnect/main_dataconnect.dart

パッケージは、firebase_data_connectを利用しています。

https://pub.dev/packages/firebase_data_connect

# 3. Data Connectを構成する

FirebaseコンソールからData Connectを構成していきます。

![](https://storage.googleapis.com/zenn-user-upload/427829866e1b-20241013.png =300x)

`Crate new Cloud SQL instance`を選びました。使用しているFirebaseのプロジェクトはSpark PlanだったのですがBlaze Planが必須のようでしたのでアップグレードしました。

![](https://storage.googleapis.com/zenn-user-upload/220ca81873c9-20241013.png =300x)

Cloud SQLのインスタンスIDやデータベース名を入力していきます。

![](https://storage.googleapis.com/zenn-user-upload/0fe5e8b577f1-20241013.png =300x)

サービスIDを入力してSubmitすると、

![](https://storage.googleapis.com/zenn-user-upload/fce1df53d228-20241013.png =300x)

以下のようにData Connectが使用できるようになりました。

![](https://storage.googleapis.com/zenn-user-upload/766f7a98f68b-20241013.png =300x)

Google CloudコンソールのCloud SQLからもデータベースが作成されたことが確認できました。

![](https://storage.googleapis.com/zenn-user-upload/549766f69ad6-20241013.png =300x)

# 4. スキーマを構成する

`Set up an environment with our extention for Visual Studio Code`とあります。
拡張機能のこのあたりを使用するとセットアップできそうです。

![](https://storage.googleapis.com/zenn-user-upload/197ce58c3f17-20241013.png =300x)

ただ、私はVSCodeをあまり使っていない(JetBrains派なので & JetBrainsの拡張機能はまだ無さそう)ということもあって、コマンドで進めてみることにしました。

## 4.1 firebase init dataconnect

```zsh
% firebase init dataconnect                                                  

     ######## #### ########  ######## ########     ###     ######  ########
     ##        ##  ##     ## ##       ##     ##  ##   ##  ##       ##
     ######    ##  ########  ######   ########  #########  ######  ######
     ##        ##  ##    ##  ##       ##     ## ##     ##       ## ##
     ##       #### ##     ## ######## ########  ##     ##  ######  ########

You're about to initialize a Firebase project in this directory:

  /Users/osaki/github/motucraft/firebase_playground

Before we get started, keep in mind:

  * You are initializing within an existing Firebase project directory


=== Project Setup

First, let's associate this project directory with a Firebase project.
You can create multiple project aliases by running firebase use --add, 
but for now we'll just set up a default project.

i  Using project fir-playground-5015e (firebase-playground)

=== Dataconnect Setup
i  dataconnect: ensuring required API firebasedataconnect.googleapis.com is enabled...
✔  dataconnect: required API firebasedataconnect.googleapis.com is enabled
i  dataconnect: ensuring required API sqladmin.googleapis.com is enabled...
✔  dataconnect: required API sqladmin.googleapis.com is enabled
i  dataconnect: ensuring required API compute.googleapis.com is enabled...
✔  dataconnect: required API compute.googleapis.com is enabled
? Your project already has existing services. Which would you like to set up local files for? asia-northeast1/playground-data-connect
✔  Wrote dataconnect/dataconnect.yaml
✔  Wrote dataconnect/schema/schema.gql
✔  Wrote dataconnect/connector/connector.yaml
✔  Wrote dataconnect/connector/mutations.gql
✔  Wrote dataconnect/connector/queries.gql
✔  Detected FLUTTER app in directory /Users/osaki/github/motucraft/firebase_playground
? Which connector do you want set up a generated SDK for? playground-data-connect/default
i  Wrote new config to /Users/osaki/github/motucraft/firebase_playground/dataconnect/connector/connector.yaml
I1013 21:10:30.517916   50323 codegen.go:82] [connector "default" dartSdk] Generating sources into "/Users/osaki/github/motucraft/firebase_playground/dataconnect-generated/dart/default_connector"
I1013 21:10:30.519871   50323 dartgen.go:575] Started Dart code generation for connector default
I1013 21:10:30.529082   50323 generate.go:40] Generated all sources. Writing them to disk...
I1013 21:10:30.530671   50323 collector.go:107] connector "default" dartSdk wrote into "/Users/osaki/github/motucraft/firebase_playground/dataconnect-generated/dart/default_connector"
Generated sources: create_movie.dart [2920B] create_movie_metadata.dart [4453B] list_movies.dart [2386B] get_movie_by_id.dart [4742B] default.dart [1376B] 

i  Generated SDK code for default
✔  If you'd like to add more generated SDKs to your app your later, run firebase init dataconnect:sdk again

i  Writing configuration info to firebase.json...
i  Writing project information to .firebaserc...

✔  Firebase initialization complete!
```

`firebase init dataconnect`を実行すると、プロジェクト直下に`dataconnect`ディレクトリ、`dataconnect-generated`ディレクトリが生成されました。
`dataconnect-generated`ディレクトリについて、以下はテーブルを作成してからの結果なので複数のdartファイルが生成されているように見えますが、初回は`default.dart`だけが生成されました。

```zsh
% tree dataconnect dataconnect-generated 
dataconnect
├── connector
│   ├── mutations.gql
│   └── queries.gql
├── dataconnect.yaml
└── schema
    └── schema.gql
dataconnect-generated
└── dart
    └── default_connector
        ├── create_movie.dart
        ├── create_movie_metadata.dart
        ├── default.dart
        ├── get_movie_by_id.dart
        └── list_movies.dart

6 directories, 10 files
```

`dataconnect-generated`はプロジェクト直下ではなくlibディレクトリ配下に配置したいため、`connector.yaml`の`outputDir`を編集しておくのが良さそうです。

```yaml
connectorId: default
generate:
  dartSdk:
    outputDir: ../../lib/dataconnect/dataconnect-generated/dart/default_connector
    package: default_connector
```

[この辺りにのドキュメント](https://firebase.google.com/docs/data-connect/flutter-sdk?_gl=1*wxtk31*_up*MQ..*_ga*NTA2OTY2NTI2LjE3Mjg4MjM0NzE.*_ga_CW55HF8NVT*MTcyODgyMzQ3MC4xLjAuMTcyODgyMzQ3MC4wLjAuMA..#generate-flutter) に説明があります。

修正したら、`firebase dataconnect:sdk:generate`を実行しておきます。`firebase dataconnect:sdk:generate --watch`で変更を監視するという方法もあるようです。

## 4.2 schema.gql を編集してテーブルを定義する

以下のように`Movie`テーブル、`MovieMetadata`テーブルを定義しました。[こちらの内容](https://firebase.google.com/docs/data-connect/quickstart-local#create_a_schema) です。

```graphql
type Movie @table {
  # The below parameter values are generated by default with @table, and can be edited manually.
  # implies directive `@col(name: "movie_id")`, generating a column name
  id: UUID! @default(expr: "uuidV4()")
  title: String!
  imageUrl: String!
  genre: String
}

type MovieMetadata @table {
  # @unique indicates a 1-1 relationship
  movie: Movie! @unique
  # movieId: UUID <- this is created by the above reference
  rating: Float
  releaseYear: Int
  description: String
}
```

## 4.3 query/mutation を定義する

- queries.gql
```graphql
query ListMovies @auth(level: PUBLIC) {
  movies {
    id
    title
    imageUrl
    genre
  }
}

query GetMovieById($id: UUID!) @auth(level: PUBLIC) {
  movie(id: $id) {
    id
    title
    imageUrl
    genre
    metadata: movieMetadata_on_movie {
      rating
      releaseYear
      description
    }
  }
}
```

- mutations.gql
```graphql
mutation CreateMovie(
  $title: String!
  $genre: String!
  $imageUrl: String!
) @auth(level: PUBLIC) {
  movie_insert(
    data: {
      title: $title
      genre: $genre
      imageUrl: $imageUrl
    }
  )
}

mutation CreateMovieMetadata(
  $movieId: UUID!
  $releaseYear: Int
  $description: String
  $rating: Float
) @auth(level: PUBLIC) {
  movieMetadata_insert(
    data: {
      movieId: $movieId
      releaseYear: $releaseYear
      description: $description
      rating: $rating
    }
  )
}
```

今回はお試しのため認証も行いませんので、mutationには`@auth(level: PUBLIC)`を付けています。

[ドキュメントはこの辺り](https://firebase.google.com/docs/data-connect/authorization-and-security#authorize_client_queries_and_mutations) が該当します。感覚としては、Firestoreのセキュリティールールと通ずるものがありそうだなと捉えました。まぁ、同じFirebaseのサービスですからね。


## 4.4 スキーマをデプロイする

以下のようにデプロイしました。

```zsh
firebase deploy --only dataconnect --project fir-playground-5015e
(node:56412) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)

=== Deploying to 'fir-playground-5015e'...

i  deploying dataconnect
i  dataconnect: ensuring required API firebasedataconnect.googleapis.com is enabled...
✔  dataconnect: required API firebasedataconnect.googleapis.com is enabled
i  dataconnect: ensuring required API sqladmin.googleapis.com is enabled...
✔  dataconnect: required API sqladmin.googleapis.com is enabled
i  dataconnect: ensuring required API compute.googleapis.com is enabled...
✔  dataconnect: required API compute.googleapis.com is enabled
i  dataconnect: Preparing to deploy
i  dataconnect: Successfully prepared schema and connectors
i  dataconnect: Checking for CloudSQL resources...
i  dataconnect: Found existing instance playground-cloud-sql.
i  dataconnect: Found existing database playground-database.
i  dataconnect: Deploying Data Connect schemas...
i  dataconnect: Schemas deployed.
i  dataconnect: Deploying connectors...
✔  dataconnect: Deployed connector projects/fir-playground-5015e/locations/asia-northeast1/services/playground-data-connect/connectors/default
i  dataconnect: Connectors deployed.
✔  dataconnect: Deployment complete! View your deployed schema and connectors at https://console.firebase.google.com/project/fir-playground-5015e/dataconnect

✔  Deploy complete!

Project Console: https://console.firebase.google.com/project/fir-playground-5015e/overview
```

これにより、以下のようにテーブルが作成されたことを確認できました。

![](https://storage.googleapis.com/zenn-user-upload/fb7c7595922a-20241013.png =300x)

このように、コンソール上からqueryを書いてDBアクセスすることもできます。

![](https://storage.googleapis.com/zenn-user-upload/c7e4f94703b8-20241013.png =300x)

# 5. [firebase_data_connect](https://pub.dev/packages/firebase_data_connect) を利用してコードを書く。

まだEarly Accessということもあってあまり情報が無いのですが、[Use generated Flutter SDKs](https://firebase.google.com/docs/data-connect/flutter-sdk?_gl=1*1qx80yr*_up*MQ..*_ga*NzA2MzI0ODE4LjE3Mjg4MjEyMzk.*_ga_CW55HF8NVT*MTcyODgyMTIzOS4xLjAuMTcyODgyMTIzOS4wLjAuMA..)を参照しながら実装していきました。

[Subscribe to changes](https://firebase.google.com/docs/data-connect/flutter-sdk?_gl=1*1qx80yr*_up*MQ..*_ga*NzA2MzI0ODE4LjE3Mjg4MjEyMzk.*_ga_CW55HF8NVT*MTcyODgyMTIzOS4xLjAuMTcyODgyMTIzOS4wLjAuMA..#subscribing-changes)という説明がありましたので、以下のように`QueryRef`を提供するProviderを用意しておき、mutation実行後に`await ref.read(listMovieRefProvider).execute()`を実行することでサブスクライブさせています。

```dart
@riverpod
QueryRef<ListMoviesData, void> listMovieRef(ListMovieRefRef ref) {
  return DefaultConnector.instance.listMovies().ref();
}

@riverpod
Stream<QueryResult<ListMoviesData, void>> movies(MoviesRef ref) {
  final listRef = ref.watch(listMovieRefProvider);
  return listRef.subscribe();
}
```

ただし、このSubscribeはリアルタイム更新ではありません。どうやら、[リアルタイム更新はまだ存在しない](https://firebase.uservoice.com/forums/948424-general/suggestions/48434600-realtime-query-updates) ようですね。

投票しておきましょう。

![](https://storage.googleapis.com/zenn-user-upload/99a9fac00ab2-20241013.png =300x)

# 6. おわりに

Firebase Data Connectをお試しして雰囲気を掴むことができました。
RDBで作られた既存システムをドキュメントベースのDB(Firestore)へ移行するのはとても大変だと思いますが、Cloud SQL(PostgreSQL)であれば選択肢が広がりそうです。

今回のお試しの中で、リアルタイム更新と同じく「あれ？これどうするの？？」と思ったのがトランザクションです。ドキュメントや、[firebase_data_connectのAPI reference](https://pub.dev/documentation/firebase_data_connect/latest/)も探してみたのですが記載は無さそうでした。

この辺りはまだEAだからということでしょうか。GAされたらまた確認してみようと思います。
