---
title: "【Supabase】GraphQLのクエリと無限スクロール (with infinite_scroll_pagination)"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "supabase", "GraphQL", "ferry", "flutter_hooks"]
published: true
---

# 1. はじめに

https://zenn.dev/motu2119/articles/supabase-graphql-query-20240504

こちらの記事の続きです。 前回は、GraphQLクライアント [ferry](https://pub.dev/packages/ferry) を利用してクエリすることを行いました。

ただ、以下のSupabase公式ドキュメントにあるとおり、

https://supabase.com/docs/guides/graphql/configuration#max-rows

> The default page size for collections is 30 entries. To adjust the number of entries on each page, set a max_rows directive on the relevant schema entity.

ということで、前回の記事ではcountriesテーブルのデータを全件取得できていませんでした。

`max_rows`を増やせば全件取得することもできますが、今回はアプリでよく見かける無限スクロールを実装してみようと思います。GraphQLでクエリしつつ、[infinite_scroll_pagination](https://pub.dev/packages/infinite_scroll_pagination)を利用して無限スクロールさせてみます。

# 2. 挙動

以下のような挙動になります。リストを下へスクロールしていくと新たにクエリが発行され、リストへレスポンスデータが追加されていきます。最終的には全件（ここでは100件）全て表示されます。

![](https://storage.googleapis.com/zenn-user-upload/aea510f1f9c5-20240505.gif)

# 3. 前提事項

Supabase上のテーブルや、GraphQLのスキーマは以前の記事（以下）で用意したものをそのまま使用しています。

https://zenn.dev/motu2119/articles/supabase-select-countries-20240504

https://zenn.dev/motu2119/articles/supabase-graphql-query-20240504

# 4. コード

## 4.1 [PagingController](https://pub.dev/documentation/infinite_scroll_pagination/latest/infinite_scroll_pagination/PagingController-class.html)

`infinite_scroll_pagination`の肝は、`PagingController`かなと思います。[flutter_hooks](https://pub.dev/packages/flutter_hooks)を利用して、以下のようにカスタムフックを用意しました。（詳細は、[How to create a hook](https://pub.dev/packages/flutter_hooks#how-to-create-a-hook)参照。）

https://github.com/motucraft/supabase_playground/blob/main/lib/hooks/use_paging_controller.dart

このカスタムフックにより、`PagingController`のインスタンスの管理をシンプルにでき、フックを呼び出すだけでページネーションの設定とデータのロードが可能になります。

```dart
  useEffect(() {
    listener(PageKeyType pageKey) => onPageRequest?.call(pageKey, controller);
    controller.addPageRequestListener(listener);
    return () => controller.removePageRequestListener(listener);
  }, [onPageRequest]);
```

この`useEffect`フックにて、ページリクエストリスナーを設定し、ページリクエストがあるたびに指定されたonPageRequestコールバックを呼び出しています。リスナーの追加とクリーンアップを行っています。

# 4.2 ページングしながらGraphQLクエリ

あとは、`infinite_scroll_pagination`の使用法のとおりだと思いますので、あまり説明することが無いのですが、以下のあたりで前回用意した`genericStreamClientProvider`を用いてクエリしています。

https://github.com/motucraft/supabase_playground/blob/main/lib/main_infinite_scroll.dart#L53

`builder.first = 30`ということで、今回は30件ずつ取得するようにしてみました。

```dart
if (pageKey != null) {
  builder.after = GCursorBuilder()..value = pageKey;
}
```

上記はカーソルを設定している箇所です。クエリの応答内の`pageInfo`に`endCursor`がありますので，この値を次のリクエストの`after`へ指定することになります。初回は存在しません（先頭から取得する）ので指定しません。
Paginationに関する説明は、以下ドキュメントに記載があります。

https://supabase.com/docs/guides/graphql/api#pagination

# 5. DevToolsでレスポンスを確認

クエリは4回発行されました。countriesテーブルには100レコード存在しており、30件ずつ取得するので想定どおりです。4回目のクエリは`"hasNextPage": false`となっており次のページが存在しないことを表しています。

:::details 1回目
```json
{
  "data": {
    "__typename": "Query",
    "countriesCollection": {
      "edges": [
        {
          "node": {
            "id": 1,
            "name": "United States",
            "__typename": "countries"
          },
          "cursor": "WzFd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 2,
            "name": "Canada",
            "__typename": "countries"
          },
          "cursor": "WzJd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 3,
            "name": "Mexico",
            "__typename": "countries"
          },
          "cursor": "WzNd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 4,
            "name": "Japan",
            "__typename": "countries"
          },
          "cursor": "WzRd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 5,
            "name": "United Kingdom",
            "__typename": "countries"
          },
          "cursor": "WzVd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 6,
            "name": "France",
            "__typename": "countries"
          },
          "cursor": "WzZd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 7,
            "name": "Germany",
            "__typename": "countries"
          },
          "cursor": "Wzdd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 8,
            "name": "Italy",
            "__typename": "countries"
          },
          "cursor": "Wzhd",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 9,
            "name": "Spain",
            "__typename": "countries"
          },
          "cursor": "Wzld",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 10,
            "name": "Australia",
            "__typename": "countries"
          },
          "cursor": "WzEwXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 11,
            "name": "New Zealand",
            "__typename": "countries"
          },
          "cursor": "WzExXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 12,
            "name": "India",
            "__typename": "countries"
          },
          "cursor": "WzEyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 13,
            "name": "China",
            "__typename": "countries"
          },
          "cursor": "WzEzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 14,
            "name": "Brazil",
            "__typename": "countries"
          },
          "cursor": "WzE0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 15,
            "name": "Argentina",
            "__typename": "countries"
          },
          "cursor": "WzE1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 16,
            "name": "Chile",
            "__typename": "countries"
          },
          "cursor": "WzE2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 17,
            "name": "South Africa",
            "__typename": "countries"
          },
          "cursor": "WzE3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 18,
            "name": "Egypt",
            "__typename": "countries"
          },
          "cursor": "WzE4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 19,
            "name": "Nigeria",
            "__typename": "countries"
          },
          "cursor": "WzE5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 20,
            "name": "Kenya",
            "__typename": "countries"
          },
          "cursor": "WzIwXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 21,
            "name": "Russia",
            "__typename": "countries"
          },
          "cursor": "WzIxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 22,
            "name": "South Korea",
            "__typename": "countries"
          },
          "cursor": "WzIyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 23,
            "name": "Thailand",
            "__typename": "countries"
          },
          "cursor": "WzIzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 24,
            "name": "Vietnam",
            "__typename": "countries"
          },
          "cursor": "WzI0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 25,
            "name": "Indonesia",
            "__typename": "countries"
          },
          "cursor": "WzI1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 26,
            "name": "Saudi Arabia",
            "__typename": "countries"
          },
          "cursor": "WzI2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 27,
            "name": "Turkey",
            "__typename": "countries"
          },
          "cursor": "WzI3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 28,
            "name": "Sweden",
            "__typename": "countries"
          },
          "cursor": "WzI4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 29,
            "name": "Norway",
            "__typename": "countries"
          },
          "cursor": "WzI5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 30,
            "name": "Finland",
            "__typename": "countries"
          },
          "cursor": "WzMwXQ==",
          "__typename": "countriesEdge"
        }
      ],
      "pageInfo": {
        "endCursor": "WzMwXQ==",
        "__typename": "PageInfo",
        "hasNextPage": true,
        "startCursor": "WzFd",
        "hasPreviousPage": false
      },
      "__typename": "countriesConnection"
    }
  }
}
```
:::

:::details 2回目
```json
{
  "data": {
    "__typename": "Query",
    "countriesCollection": {
      "edges": [
        {
          "node": {
            "id": 31,
            "name": "Portugal",
            "__typename": "countries"
          },
          "cursor": "WzMxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 32,
            "name": "Ireland",
            "__typename": "countries"
          },
          "cursor": "WzMyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 33,
            "name": "Greece",
            "__typename": "countries"
          },
          "cursor": "WzMzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 34,
            "name": "Hungary",
            "__typename": "countries"
          },
          "cursor": "WzM0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 35,
            "name": "Switzerland",
            "__typename": "countries"
          },
          "cursor": "WzM1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 36,
            "name": "Austria",
            "__typename": "countries"
          },
          "cursor": "WzM2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 37,
            "name": "Netherlands",
            "__typename": "countries"
          },
          "cursor": "WzM3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 38,
            "name": "Belgium",
            "__typename": "countries"
          },
          "cursor": "WzM4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 39,
            "name": "Poland",
            "__typename": "countries"
          },
          "cursor": "WzM5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 40,
            "name": "Czech Republic",
            "__typename": "countries"
          },
          "cursor": "WzQwXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 41,
            "name": "Slovakia",
            "__typename": "countries"
          },
          "cursor": "WzQxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 42,
            "name": "Denmark",
            "__typename": "countries"
          },
          "cursor": "WzQyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 43,
            "name": "Iceland",
            "__typename": "countries"
          },
          "cursor": "WzQzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 44,
            "name": "Croatia",
            "__typename": "countries"
          },
          "cursor": "WzQ0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 45,
            "name": "Bulgaria",
            "__typename": "countries"
          },
          "cursor": "WzQ1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 46,
            "name": "Romania",
            "__typename": "countries"
          },
          "cursor": "WzQ2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 47,
            "name": "Ukraine",
            "__typename": "countries"
          },
          "cursor": "WzQ3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 48,
            "name": "Israel",
            "__typename": "countries"
          },
          "cursor": "WzQ4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 49,
            "name": "Lebanon",
            "__typename": "countries"
          },
          "cursor": "WzQ5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 50,
            "name": "United Arab Emirates",
            "__typename": "countries"
          },
          "cursor": "WzUwXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 51,
            "name": "Kuwait",
            "__typename": "countries"
          },
          "cursor": "WzUxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 52,
            "name": "Qatar",
            "__typename": "countries"
          },
          "cursor": "WzUyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 53,
            "name": "Bahrain",
            "__typename": "countries"
          },
          "cursor": "WzUzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 54,
            "name": "Oman",
            "__typename": "countries"
          },
          "cursor": "WzU0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 55,
            "name": "Morocco",
            "__typename": "countries"
          },
          "cursor": "WzU1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 56,
            "name": "Algeria",
            "__typename": "countries"
          },
          "cursor": "WzU2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 57,
            "name": "Tunisia",
            "__typename": "countries"
          },
          "cursor": "WzU3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 58,
            "name": "Libya",
            "__typename": "countries"
          },
          "cursor": "WzU4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 59,
            "name": "Senegal",
            "__typename": "countries"
          },
          "cursor": "WzU5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 60,
            "name": "Ivory Coast",
            "__typename": "countries"
          },
          "cursor": "WzYwXQ==",
          "__typename": "countriesEdge"
        }
      ],
      "pageInfo": {
        "endCursor": "WzYwXQ==",
        "__typename": "PageInfo",
        "hasNextPage": true,
        "startCursor": "WzMxXQ==",
        "hasPreviousPage": true
      },
      "__typename": "countriesConnection"
    }
  }
}
```
:::

:::details 3回目
```json
{
  "data": {
    "__typename": "Query",
    "countriesCollection": {
      "edges": [
        {
          "node": {
            "id": 61,
            "name": "Ghana",
            "__typename": "countries"
          },
          "cursor": "WzYxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 62,
            "name": "Cameroon",
            "__typename": "countries"
          },
          "cursor": "WzYyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 63,
            "name": "Tanzania",
            "__typename": "countries"
          },
          "cursor": "WzYzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 64,
            "name": "Uganda",
            "__typename": "countries"
          },
          "cursor": "WzY0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 65,
            "name": "Ethiopia",
            "__typename": "countries"
          },
          "cursor": "WzY1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 66,
            "name": "Bangladesh",
            "__typename": "countries"
          },
          "cursor": "WzY2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 67,
            "name": "Pakistan",
            "__typename": "countries"
          },
          "cursor": "WzY3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 68,
            "name": "Sri Lanka",
            "__typename": "countries"
          },
          "cursor": "WzY4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 69,
            "name": "Nepal",
            "__typename": "countries"
          },
          "cursor": "WzY5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 70,
            "name": "Maldives",
            "__typename": "countries"
          },
          "cursor": "WzcwXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 71,
            "name": "Myanmar",
            "__typename": "countries"
          },
          "cursor": "WzcxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 72,
            "name": "Laos",
            "__typename": "countries"
          },
          "cursor": "WzcyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 73,
            "name": "Cambodia",
            "__typename": "countries"
          },
          "cursor": "WzczXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 74,
            "name": "Malaysia",
            "__typename": "countries"
          },
          "cursor": "Wzc0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 75,
            "name": "Philippines",
            "__typename": "countries"
          },
          "cursor": "Wzc1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 76,
            "name": "Singapore",
            "__typename": "countries"
          },
          "cursor": "Wzc2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 77,
            "name": "Brunei",
            "__typename": "countries"
          },
          "cursor": "Wzc3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 78,
            "name": "Mongolia",
            "__typename": "countries"
          },
          "cursor": "Wzc4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 79,
            "name": "Kazakhstan",
            "__typename": "countries"
          },
          "cursor": "Wzc5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 80,
            "name": "Uzbekistan",
            "__typename": "countries"
          },
          "cursor": "WzgwXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 81,
            "name": "Turkmenistan",
            "__typename": "countries"
          },
          "cursor": "WzgxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 82,
            "name": "Kyrgyzstan",
            "__typename": "countries"
          },
          "cursor": "WzgyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 83,
            "name": "Tajikistan",
            "__typename": "countries"
          },
          "cursor": "WzgzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 84,
            "name": "Belarus",
            "__typename": "countries"
          },
          "cursor": "Wzg0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 85,
            "name": "Estonia",
            "__typename": "countries"
          },
          "cursor": "Wzg1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 86,
            "name": "Latvia",
            "__typename": "countries"
          },
          "cursor": "Wzg2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 87,
            "name": "Lithuania",
            "__typename": "countries"
          },
          "cursor": "Wzg3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 88,
            "name": "Slovenia",
            "__typename": "countries"
          },
          "cursor": "Wzg4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 89,
            "name": "Serbia",
            "__typename": "countries"
          },
          "cursor": "Wzg5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 90,
            "name": "Bosnia and Herzegovina",
            "__typename": "countries"
          },
          "cursor": "WzkwXQ==",
          "__typename": "countriesEdge"
        }
      ],
      "pageInfo": {
        "endCursor": "WzkwXQ==",
        "__typename": "PageInfo",
        "hasNextPage": true,
        "startCursor": "WzYxXQ==",
        "hasPreviousPage": true
      },
      "__typename": "countriesConnection"
    }
  }
}
```
:::

:::details 4回目
```json
{
  "data": {
    "__typename": "Query",
    "countriesCollection": {
      "edges": [
        {
          "node": {
            "id": 91,
            "name": "Macedonia",
            "__typename": "countries"
          },
          "cursor": "WzkxXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 92,
            "name": "Montenegro",
            "__typename": "countries"
          },
          "cursor": "WzkyXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 93,
            "name": "Georgia",
            "__typename": "countries"
          },
          "cursor": "WzkzXQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 94,
            "name": "Armenia",
            "__typename": "countries"
          },
          "cursor": "Wzk0XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 95,
            "name": "Azerbaijan",
            "__typename": "countries"
          },
          "cursor": "Wzk1XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 96,
            "name": "Bolivia",
            "__typename": "countries"
          },
          "cursor": "Wzk2XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 97,
            "name": "Paraguay",
            "__typename": "countries"
          },
          "cursor": "Wzk3XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 98,
            "name": "Peru",
            "__typename": "countries"
          },
          "cursor": "Wzk4XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 99,
            "name": "Uruguay",
            "__typename": "countries"
          },
          "cursor": "Wzk5XQ==",
          "__typename": "countriesEdge"
        },
        {
          "node": {
            "id": 100,
            "name": "Venezuela",
            "__typename": "countries"
          },
          "cursor": "WzEwMF0=",
          "__typename": "countriesEdge"
        }
      ],
      "pageInfo": {
        "endCursor": "WzEwMF0=",
        "__typename": "PageInfo",
        "hasNextPage": false,
        "startCursor": "WzkxXQ==",
        "hasPreviousPage": true
      },
      "__typename": "countriesConnection"
    }
  }
}
```
:::

# 6. おわりに

以下のパッケージのおかげで、かなり簡単にGraphQLでクエリし、無限スクロールを実装することができました。ありがたや。

- [ferry](https://pub.dev/packages/ferry)
- [infinite_scroll_pagination](https://pub.dev/packages/infinite_scroll_pagination)
- [riverpod](https://pub.dev/packages/riverpod/versions/3.0.0-dev.3)
- [flutter_hooks](https://pub.dev/packages/flutter_hooks)

今回クエリしてみて気づいたのですが、Supabaseのクエリはかなり応答が速いですね。使用したデータ量は少ないものの、各クエリが2ケタミリ秒程度で毎回応答されてくるので、いいね👍と思いました（初回のコネクションを張るのには多少時間を要するのでしょうか？）。ただ、速すぎてローディングの表示が確認できなかったため、あえて以下の箇所で遅延させています...

https://github.com/motucraft/supabase_playground/blob/main/lib/main_infinite_scroll.dart#L70

それから、現時点ではGraphQLのSubscriptionはサポートされていないようです。以下のとおり、Feature Requestは出ているようです。リアルタイム更新が必要な場合は、`Supabase.instance.client`を利用することになると思います。

https://github.com/supabase/pg_graphql/issues/17
