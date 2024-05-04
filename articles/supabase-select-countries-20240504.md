---
title: "【Supabase】Quickstarts (with Riverpod)"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "supabase"]
published: true
---

# 1. はじめに

以前から[Supabase](https://supabase.com/)が気になっているものの、なかなか業務利用することができませんので、個人的に少しずつ触っていこうと思います。
まずは、以下のUse Supabase with Flutterに従って、countriesテーブルをselectしてみることから始めようと思います。

https://supabase.com/docs/guides/getting-started/quickstarts/flutter

# 2. 挙動

以下のように、countriesテーブルの国名を一覧表示します。リロードボタンをタップすると、selectを再実行します。DevToolsからもリクエストが行われていることを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/e09e72e6cfa4-20240504.gif)

# 3. テーブル作成

Quickstartsに記載されているcountriesテーブルを作成しつつ、SQL Editorから100件insertしておきました。

![](https://storage.googleapis.com/zenn-user-upload/7edfc52de978-20240504.png)

# 4. コード

selectするだけなので単純です。以下、コードの全量です。
https://github.com/motucraft/supabase_playground/blob/main/lib/main_select_countries.dart

`Supabase.initialize`の箇所で、[flutter_dotenv](https://pub.dev/packages/flutter_dotenv)を使用しています。
Project URLやProject API keysは、Supabase ConsoleのAPI Settingsから確認することができます。

```dart
  final url = dotenv.env['SUPABASE_URL'];
  final anonKey = dotenv.env['SUPABASE_ANON_KEY'];

  if (url?.isNotEmpty != true || anonKey?.isNotEmpty != true) {
    throw StateError('Supabase URL and Anon Key must be provided.');
  }

  await Supabase.initialize(url: url!, anonKey: anonKey!);
```

Countriesウィジェットでは、以下のProviderをwatchしてselect結果を表示しています。リロード時には、このProviderをinvalidateすることで再取得しています。

```dart
@riverpod
class CountriesNotifier extends _$CountriesNotifier {
  @override
  Future<List<PostgrestMap>> build() =>
      Supabase.instance.client.from('countries').select();
}
```

# 5. おわりに

まずは単純にselectして応答を表示することができました。次はGraphQLでQueryしてみたいと思います。
