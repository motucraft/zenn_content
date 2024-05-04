---
title: "【Supabase】GraphQL Query (with Ferry and Riverpod 3.0)"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "supabase", "GraphQL"]
published: false
---

# 1. はじめに

https://zenn.dev/motu2119/articles/supabase-select-countries-20240504

こちらの記事の続きです。今回は、前回作成したcountriesテーブルのデータをGraphQLでQueryしたいと思います。GraphQLのクライアントには、以下のferryを利用します。

https://pub.dev/packages/ferry

https://ferrygraphql.com/docs/

# 2. 挙動

アプリの見た目は前回と同様ですが、以下のようにDevToolsからGraphQLのQueryが成功していることを確認できました。

![](https://storage.googleapis.com/zenn-user-upload/4084be1a27d8-20240504.gif)

# 3. GraphQL

## 3.1 スキーマの入手

クライアントを実装するために、まずはGraphQLのスキーマを入手します。方法はいくつかあると思いますが、私はApolloのRover CLIを利用しました。npmでグローバルインストールしておきます。
https://www.apollographql.com/docs/rover/getting-started#global-install

```shell
npm install -g @apollo/rover
```

以下のようにroverにてスキーマを入手し、schema.graphqlへリダイレクトしました。

```shell
rover graph introspect <YOUR_SUPABASE_URL> --header "apiKey: <YOUR_SUPABASE_ANON_KEY>" > schema.graphql
```

入手できたスキーマは以下のとおりです。

:::details schema.graphql
```graphql
schema {
  query: Query
  mutation: Mutation
}
"A high precision floating point value represented as a string"
scalar BigFloat
"An arbitrary size integer represented as a string"
scalar BigInt
"An opaque string using for tracking a position in results during pagination"
scalar Cursor
"A date wihout time information"
scalar Date
"A date and time"
scalar Datetime
"A Javascript Object Notation value serialized as a string"
scalar JSON
"Any type not handled by the type system"
scalar Opaque
"A time without date information"
scalar Time
"A universally unique identifier"
scalar UUID
"The root type for creating and mutating data"
type Mutation {
  "Deletes zero or more records from the `countries` collection"
  deleteFromcountriesCollection(
    "Restricts the mutation's impact to records matching the criteria"
    filter: countriesFilter,
    "The maximum number of records in the collection permitted to be affected"
    atMost: Int! = 1
  ): countriesDeleteResponse!
  "Adds one or more `countries` records to the collection"
  insertIntocountriesCollection(objects: [countriesInsertInput!]!): countriesInsertResponse
  "Updates zero or more records in the `countries` collection"
  updatecountriesCollection(
    "Fields that are set will be updated for all records matching the `filter`"
    set: countriesUpdateInput!,
    "Restricts the mutation's impact to records matching the criteria"
    filter: countriesFilter,
    "The maximum number of records in the collection permitted to be affected"
    atMost: Int! = 1
  ): countriesUpdateResponse!
}
type PageInfo {
  endCursor: String
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
}
"The root type for querying data"
type Query {
  "A pagable collection of type `countries`"
  countriesCollection(
    "Query the first `n` records in the collection"
    first: Int,
    "Query the last `n` records in the collection"
    last: Int,
    "Query values in the collection before the provided cursor"
    before: Cursor,
    "Query values in the collection after the provided cursor"
    after: Cursor,
    "Skip n values from the after cursor. Alternative to cursor pagination. Backward pagination not supported."
    offset: Int,
    "Filters to apply to the results set when querying from the collection"
    filter: countriesFilter,
    "Sort order to apply to the collection"
    orderBy: [countriesOrderBy!]
  ): countriesConnection
  "Retrieve a record by its `ID`"
  node(
    "The record's `ID`"
    nodeId: ID!
  ): Node
}
type countries implements Node {
  "Globally Unique Record Identifier"
  nodeId: ID!
  id: Int!
  name: String!
}
type countriesConnection {
  edges: [countriesEdge!]!
  pageInfo: PageInfo!
}
type countriesDeleteResponse {
  "Count of the records impacted by the mutation"
  affectedCount: Int!
  "Array of records impacted by the mutation"
  records: [countries!]!
}
type countriesEdge {
  cursor: String!
  node: countries!
}
type countriesInsertResponse {
  "Count of the records impacted by the mutation"
  affectedCount: Int!
  "Array of records impacted by the mutation"
  records: [countries!]!
}
type countriesUpdateResponse {
  "Count of the records impacted by the mutation"
  affectedCount: Int!
  "Array of records impacted by the mutation"
  records: [countries!]!
}
interface Node {
  "Retrieves a record by `ID`"
  nodeId: ID!
}
enum FilterIs {
  NULL
  NOT_NULL
}
"Defines a per-field sorting order"
enum OrderByDirection {
  "Ascending order, nulls first"
  AscNullsFirst
  "Ascending order, nulls last"
  AscNullsLast
  "Descending order, nulls first"
  DescNullsFirst
  "Descending order, nulls last"
  DescNullsLast
}
"""
Boolean expression comparing fields on type "BigFloat"
"""
input BigFloatFilter {
  eq: BigFloat
  gt: BigFloat
  gte: BigFloat
  in: [BigFloat!]
  is: FilterIs
  lt: BigFloat
  lte: BigFloat
  neq: BigFloat
}
"""
Boolean expression comparing fields on type "BigInt"
"""
input BigIntFilter {
  eq: BigInt
  gt: BigInt
  gte: BigInt
  in: [BigInt!]
  is: FilterIs
  lt: BigInt
  lte: BigInt
  neq: BigInt
}
"""
Boolean expression comparing fields on type "Boolean"
"""
input BooleanFilter {
  eq: Boolean
  is: FilterIs
}
"""
Boolean expression comparing fields on type "Date"
"""
input DateFilter {
  eq: Date
  gt: Date
  gte: Date
  in: [Date!]
  is: FilterIs
  lt: Date
  lte: Date
  neq: Date
}
"""
Boolean expression comparing fields on type "Datetime"
"""
input DatetimeFilter {
  eq: Datetime
  gt: Datetime
  gte: Datetime
  in: [Datetime!]
  is: FilterIs
  lt: Datetime
  lte: Datetime
  neq: Datetime
}
"""
Boolean expression comparing fields on type "Float"
"""
input FloatFilter {
  eq: Float
  gt: Float
  gte: Float
  in: [Float!]
  is: FilterIs
  lt: Float
  lte: Float
  neq: Float
}
"""
Boolean expression comparing fields on type "ID"
"""
input IDFilter {
  eq: ID
}
"""
Boolean expression comparing fields on type "Int"
"""
input IntFilter {
  eq: Int
  gt: Int
  gte: Int
  in: [Int!]
  is: FilterIs
  lt: Int
  lte: Int
  neq: Int
}
"""
Boolean expression comparing fields on type "Opaque"
"""
input OpaqueFilter {
  eq: Opaque
  is: FilterIs
}
"""
Boolean expression comparing fields on type "String"
"""
input StringFilter {
  eq: String
  gt: String
  gte: String
  ilike: String
  in: [String!]
  iregex: String
  is: FilterIs
  like: String
  lt: String
  lte: String
  neq: String
  regex: String
  startsWith: String
}
"""
Boolean expression comparing fields on type "Time"
"""
input TimeFilter {
  eq: Time
  gt: Time
  gte: Time
  in: [Time!]
  is: FilterIs
  lt: Time
  lte: Time
  neq: Time
}
"""
Boolean expression comparing fields on type "UUID"
"""
input UUIDFilter {
  eq: UUID
  in: [UUID!]
  is: FilterIs
  neq: UUID
}
input countriesFilter {
  id: IntFilter
  name: StringFilter
  nodeId: IDFilter
  "Returns true only if all its inner filters are true, otherwise returns false"
  and: [countriesFilter!]
  "Returns true if at least one of its inner filters is true, otherwise returns false"
  or: [countriesFilter!]
  "Negates a filter"
  not: countriesFilter
}
input countriesInsertInput {
  name: String
}
input countriesOrderBy {
  id: OrderByDirection
  name: OrderByDirection
}
input countriesUpdateInput {
  name: String
}
```
:::

## 3.2 クエリ

スキーマを元に、クエリを作成します。以下のように`query Countries`を用意しました。

```graphql
query Countries($first: Int) {
  countriesCollection(first: $first) {
    edges {
      node {
        id
        name
      }
    }
    pageInfo {
      endCursor
      hasNextPage
      hasPreviousPage
      startCursor
    }
  }
}
```

そして、以下のようにlibディレクトリ内へ格納しました。

![](https://storage.googleapis.com/zenn-user-upload/fc78e7b8e1aa-20240504.png)

## 3.3 クエリをお試し

クエリが実行できるか確認してみます。Supabase ConsoleのAPI Docsから確認できます。レスポンスが取れていますね。

![](https://storage.googleapis.com/zenn-user-upload/d512b6c8f65b-20240504.png)

## 3.4 自動生成

あとは自動生成です。いつものように`build_runner`を実行するのですが、 その前に`build.yaml`が必要ですので、[Build Generated Classes](https://ferrygraphql.com/docs/codegen#build-generated-classes)を参照して設定しておきます。
私の場合は以下のように設定しておきました。まだ使用しないのですが、後に使うかも？ということでDateTimeのシリアライザ (date_time_serializer.dart) も設定しておきました。

```yaml
targets:
  $default:
    builders:
      ferry_generator|graphql_builder:
        enabled: true
        options:
          schema: supabase_playground|lib/graphql/schema.graphql
          global_enum_fallbacks: true
          type_overrides:
            Date:
              name: DateTime
            DateTime:
              name: DateTime
            JSON:
              name: JsonObject
              import: package:built_value/json_object.dart

      ferry_generator|serializer_builder:
        enabled: true
        options:
          schema: supabase_playground|lib/graphql/schema.graphql
          type_overrides:
            Date:
              name: DateTime
            DateTime:
              name: DateTime
            JSON:
              name: JsonObject
              import: package:built_value/json_object.dart
          custom_serializers:
            - name: DateTimeSerializer
              import: package:supabase_playground/graphql/serializer/date_time_serializer.dart
            - name: JsonObjectSerializer
              import: package:built_value/src/json_object_serializer.dart
      json_serializable:
        options:
          create_field_map: true
          create_per_field_to_json: true
```

`build_runner`を実行します。

```shell
flutter pub run build_runner build --delete-conflicting-outputs
```

以下のように、`__generated__`ディレクトリ内にferryのコードが生成されました。

![](https://storage.googleapis.com/zenn-user-upload/9ebaa0896c16-20240504.png)

# 4. Client

クライアントを実装していきます。

```dart
import 'package:ferry/ferry.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:gql_http_link/gql_http_link.dart';
import 'package:http/http.dart' as http;
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'client_provider.g.dart';

@Riverpod(keepAlive: true)
TypedLink client(ClientRef ref) {
  final url = '${dotenv.env['SUPABASE_URL']}/graphql/v1';
  return Client(
    link: HttpLink(
      url,
      httpClient: ApiKeyClient(http.Client()),
    ),
    defaultFetchPolicies: {
      OperationType.query: FetchPolicy.NetworkOnly,
    },
  );
}

@riverpod
class GenericStreamClient<TData, TVars>
    extends _$GenericStreamClient<TData, TVars> {
  @override
  Stream<OperationResponse<TData, TVars>> build(
      OperationRequest<TData, TVars> request) {
    return ref.watch(clientProvider).request(request);
  }
}

class ApiKeyClient extends http.BaseClient {
  final http.Client client;

  ApiKeyClient(this.client);

  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) {
    final anonKey = dotenv.env['SUPABASE_ANON_KEY'];
    if (request is http.Request && anonKey?.isNotEmpty == true) {
      request.headers['apiKey'] = anonKey!;
    }

    return client.send(request);
  }
}
```

`clientProvider`でferryのClientインスタンスを生成しています。Clientインスタンスは生き続ければ良いため、`keepAlive: true`としています。`FetchPolicy.NetworkOnly`を指定してferryのキャシュを使用しないようにしています。この辺りはキャッシュ戦略次第かと思います。

クエリする際にはSupabaseのAPIキーを指定しないと、`401 Unauthorized`ということになりますので、`ApiKeyClient`クラスにてキーを付与しています。ANONキーで良いみたいです。（私はSupabase初心者なので、service_roleのキーとの違いがよくわかっていません...調べねば）

そして、この記事のタイトルにある「and Riverpod 3.0」についてです。Riverpod 3.0ではGenericsが利用可能となっています。Riverpod 3.0は、現時点ではdevバージョンです（[3.0.0-dev.3](https://pub.dev/packages/riverpod/versions/3.0.0-dev.3)）。devブランチは更新されているのですがpublishはされておらず、数ヶ月間`3.0.0-dev.3`のまま更新がありません。そのため、Discordでもコメントしてみました。Remiさん次第ですね（3.0のstableはまだ先のようです。今年中ぐらいでしょうか？）。

https://discord.com/channels/765557403865186374/765557404766830614/1235902253303332919

以下でGenericsを使用しています。今回は単一のクエリしか使用しませんが、このようにGenericsのProviderを用意しておけば、他のクエリも同様にこのProviderを使用することができます。Genericsのおかげで、似て非なるProviderを多数生み出す必要がなくなります。

```dart
@riverpod
class GenericStreamClient<TData, TVars>
    extends _$GenericStreamClient<TData, TVars> {
  @override
  Stream<OperationResponse<TData, TVars>> build(
      OperationRequest<TData, TVars> request) {
    return ref.watch(clientProvider).request(request);
  }
}
```

# 5. Widget

Clientを用意しましたので、あとは`genericStreamClientProvider`をwatchするウィジェットを実装してクエリの結果を画面表示します。

https://github.com/motucraft/supabase_playground/blob/main/lib/main_graphql_query.dart

```dart
    final clientProvider = genericStreamClientProvider(
      GCountriesReq(
        (b) => b
          ..vars.update((builder) {
            // https://supabase.com/docs/guides/graphql/configuration#max-rows
            //   The default page size for collections is 30 entries. To adjust the number of entries on each page, set a max_rows directive on the relevant schema entity.
            builder.first = 20;
          }),
      ),
    );
```

Supabaseのデフォルトでは`The default page size for collections is 30 entries`ということのようですね。ここでは20件取得するようにしてみました。

# 6. おわりに

SupabaseにてGraphQLを利用しクエリすることができました。`builder.first = 20`としたため、全件表示することはできていません。
そこで、次回は[infinite_scroll_pagination](https://pub.dev/packages/infinite_scroll_pagination)を利用してGraphQLのQueryを行いつつ無限スクロールさせてみようと思います。
