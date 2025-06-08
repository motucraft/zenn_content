---
title: "【Flutter】Riverpod 3.0 の Offline persistence を drift で永続化する"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "riverpod", "drift"]
published: true
---

# 1. はじめに

Riverpod 3.0 の実験的新機能として、[Offline persistence (experimental)](https://riverpod.dev/ja/docs/whats_new#offline-persistence-experimental) が導入されました。この機能を使うと、Notifier プロバイダの状態をデータベースに永続化し、アプリ起動時にキャッシュから復元できます。

公式ドキュメントでは [riverpod_sqflite](https://pub.dev/packages/riverpod_sqflite) を利用し、SQLite（sqflite）を利用した永続化サンプルが掲載されていますが、

> Riverpod only includes interfaces to interact with a database. It does not include a database itself. You can use any database you want, as long as it implements the interfaces.

とあります。

インタフェースを実装すれば `You can use any database you want` ということですので、ここでは [drift](https://pub.dev/packages/drift) を利用して Offline persistence を試してみようと思います。

# 2. sqflite ベース実装の確認

まずは、`riverpod_sqflite` および [sqflite](https://pub.dev/packages/sqflite) を利用した例です。公式ドキュメントのコードを利用しています。

https://github.com/motucraft/riverpod_playground/blob/master/lib/offline_persistence/sqflite/main_sqflite.dart

SQLite のデータベースをオープンし、JSON シリアライズ／デシリアライズ機能を持つストレージアダプタを返却する Provider を用意します。

```dart
@riverpod
Future<JsonSqFliteStorage> storage(Ref ref) async {
  return JsonSqFliteStorage.open(
    join(await getDatabasesPath(), 'offline_persistence_sqflite.db'),
  );
}
```

`TodosNotifier` には、[JsonPersist](https://pub.dev/documentation/riverpod_annotation/3.0.0-dev.15/experimental_json_persist/JsonPersist-class.html) アノテーションを付与し、Notifier が持つ状態（List<Todo>）を JSON シリアライズして保存・復元する機能を有効化します。さらに、[persist](https://pub.dev/documentation/riverpod/3.0.0-dev.15/experimental_persist/Persistable/persist.html) メソッドでは、先ほど定義した `storageProvider` を通じて `JsonSqFliteStorage` のインスタンスを渡し、実際の永続化先を指定しています。

```dart
@riverpod
@JsonPersist()
class TodosNotifier extends _$TodosNotifier {
  @override
  FutureOr<List<Todo>> build() async {
    await persist(
      storage: ref.watch(storageProvider.future),
      options: const StorageOptions(cacheTime: StorageCacheTime.unsafe_forever),
    );

    return state.value ?? [];
  }

  Future<void> add(Todo todo) async {
    state = AsyncData([...await future, todo]);
  }
}
```

# 3. drift の導入

`sqflite` の例を確認できましたので、ここからは [drift](https://pub.dev/packages/drift) を利用していきます。drift のドキュメント [Getting Started](https://drift.simonbinder.eu/setup/) に従いましょう。

`pubspec.yaml` はこちらです。

https://github.com/motucraft/riverpod_playground/blob/master/pubspec.yaml

そして、以下のように `AppDatabase` を定義しました。

https://github.com/motucraft/riverpod_playground/blob/d2780bbb0ecec964d0cf03eb531e3f28e347461c/lib/offline_persistence/drift/database.dart#L20-L35

# 4. テーブル定義

このように drift のテーブルを用意しました。

https://github.com/motucraft/riverpod_playground/blob/d2780bbb0ecec964d0cf03eb531e3f28e347461c/lib/offline_persistence/drift/database.dart#L10-L18

| カラム名       | 説明                                                                                                                                                                                                     |
|:-----------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| key        | キャッシュを識別するユニークキー。Notifier ごと、あるいはデータ種別ごとに一意の文字列を設定します。                                                                                                                                                 |
| jsonValue  | 保存対象のオブジェクトを JSON 文字列化して格納します。復元時にはデシリアライズして元の型に戻します。                                                                                                                                                  |
| destroyKey | スキーマ変更や強制リセット時に、[StorageOptions.destroyKey](https://pub.dev/documentation/riverpod/3.0.0-dev.15/experimental_persist/StorageOptions/destroyKey.html) の値が変わると古いキャッシュが削除される際に利用される識別子です。                 |
| expireAt   | キャッシュの有効期限を示す日時。[StorageOptions.cacheTime](https://pub.dev/documentation/riverpod/3.0.0-dev.15/experimental_persist/StorageOptions/cacheTime.html) の設定に基づき `persist` 実装が更新し、期限切れ判定に使用します。null なら無期限です。 |
| updatedAt  | レコード挿入時に自動で `CURRENT_TIMESTAMP` がセットされる日時カラム。以降の自動更新は行われないため、必要に応じてアプリ側で上書きしてください。                                                                                                                     |


# 5. Riverpod プロバイダへの組み込み

## 5.1 DriftStorage

`DriftStorage` は、[Storage](https://pub.dev/documentation/riverpod/3.0.0-dev.15/experimental_persist/Storage-class.html) を実装します。これが、`You can use any database you want` ということですね。

https://github.com/motucraft/riverpod_playground/blob/d2780bbb0ecec964d0cf03eb531e3f28e347461c/lib/offline_persistence/drift/database.dart#L37-L88

- read
  - `key` で `JsonCacheTable` を検索し、該当レコードを取得
  - JSON 値・`destroyKey`・`expireAt` を `PersistedData` に詰めて返却
- write (upsert)
  - `cacheTime` に応じて `expireAt` を算出
  - `insertOnConflictUpdate` を利用して UPSERT
  - `updatedAt` は手動で現在時刻を設定
- delete
  - `key` のレコードを削除

## 5.2 TodosNotifier

https://github.com/motucraft/riverpod_playground/blob/d2780bbb0ecec964d0cf03eb531e3f28e347461c/lib/offline_persistence/drift/main_drift.dart#L76-L92

- persist
  `storageProvider` 経由で取得した `DriftStorage` を渡し、初回起動時の読み込みと以降の自動書き込みを有効化します。
- [StorageOptions](https://pub.dev/documentation/riverpod/3.0.0-dev.15/experimental_persist/StorageOptions-class.html)
  unsafe_forever によって無期限キャッシュとし、手動でのクリアやマイグレーション時に destroyKey を使ってリセット可能です。`cacheTime` は、`By default, state is cached offline only for 2 days.` です。
- add
  通常の Notifier と同様に state を更新するだけで、自動的に JSON 化して DB に保存される仕組みです。

# 6. 動作確認

このように、アプリのプロセスを落としてコールドスタートしても初期表示されることが確認できました。

![](https://storage.googleapis.com/zenn-user-upload/b3d202faa3d9-20250608.gif =300x)

- **コールドスタート**: アプリ起動時に以前追加した Todo が表示される
- **新規追加**: FAB で Todo を追加 → 即座にリストが更新され、アプリ再起動後も残存

# 7. おわりに

Riverpod 3.0 の Offline persistence（実験機能）について drift を使って永続化することができました。

現時点では v3.0 は dev バージョンですが、まもなく stable がリリースされるのではと思います。他の新機能や bugfix なども確認していこうと思います。そして、既存アプリの v3 へのマイグレーションも考えていく必要がありそうです。
