---
title: GraphQL
emoji: "💻"
type: "tech"
topics: ["graphql"]
published: false
---


# GraphQL とは

GraphQL は API 用のクエリ言語 (query language) とクエリを実行するサーバー側のランタイムのことであり、クライアントが必要なデータを指定することができ、必要な情報だけを１つのリクエストだけで取得することができる。

Facebook (現 Meta) が 2012 年に GraphQL を開発し始め、2015 年に仕様のドラフトとリファレンス実装がオープンソースとして公開された。その後、2018 年に GraphQL は新設された GraphQL Foundation (Linux Foundation にホストされている) に移譲された。


# 型システム (Type System) / スキーマ (Schema)

クライアントがクエリを行えるようにするために、型システムを使って API のスキーマを定義する必要がある。

GraphQL では 6 種類の型がある。
- スカラー型 (Scalars)
- 列挙型 (Enums)
- オブジェクト型 (Objects)
- インターフェース型 (Interfaces)
- ユニオン型 (Unions)
- 入力オブジェクト型 (Input Objects)

最低限、スカラー型、列挙型、オブジェクト型は理解している必要がある。

## スカラー型 (Scalars)

ビルトインスカラー型は、以下の 5 種類である。

- 整数 (`Int`)
- 浮動小数点数 (`Float`)
- 文字列 (`String`)
- 真偽値 (`Boolean`)
- 識別子 (`ID`)

独自のスカラー型も定義することが可能であり、準拠すべき仕様 / RFC を指定することができる。

```
scalar Url @specifiedBy(url: "https://tools.ietf.org/html/rfc3986")
```

## 列挙型 (Enums)

列挙型は、取りうる値を列挙する形で定義される。

```
enum Direction {
    NORTH
    EAST
    SOUTH
    WEST
}
```

## オブジェクト型 (Objects)

主に独自の型を定義するのに用いる。

以下のように、型名に続いて、フィールド名と型のペアが複数続くのが基本形である。

```
type Person {
    name: String
    age: Int
    picture: Url
}
```

フィールドには 0 個以上の引数を指定することもできる。

```
type Person {
    name: String
    picture(size: Int): Url
}
```

型の後ろに `!` をつけることで、必須のフィールドであることを示すことができる。

```
type Person {
    id: ID!
    name: String!
}
```

同じ型の値の集合として、リスト (List) を利用することもできる。

```
type Person {
    id: ID!
    name: String!
    hobby: [String]
}
```

## 操作 (Operations)

GraphQL では、クライアントは以下の 3 種類の操作をすることができる。
- Query
- Mutation
- Subscription

少なくとも Query は必ず定義されなければならない。

```
type Query {
    person(id: ID!): Person
}
```


# クエリ (Queries)

上述の通り、GraphQL では以下の 3 種類の操作がサポートされている。
- Query
- Mutation
- Subscription

## フィールド (Fields)

以下のように Query タイプに `hero` フィールドが定義されているとする。

```
type Query {
    here: Character
}
```

以下のようなクエリを使って、特定のフィールドを取得することができる。

例 1:
```
{
    hero {
        name
    }
}
```
```
{
    "data": {
        "hero": {
            "name": "R2-D2"
        }
    }
}
```

例 2:
```
{
    hero {
        name
        friends {
            name
        }
    }
}
```
```
{
    "data": {
        "hero": {
            "name": "R2-D2",
            "friends": [
                {
                    "name": "Luke Skywalker"
                },
                {
                    "name": "Han Solo"
                },
                {
                    "name": "Leia Organa"
                }
            ]
        }
    }
}
```

## 引数 (Arguments)

以下のような Query タイプが定義されていたとする。

```
type Query {
    human(id: ID!): Human
}
```

`id` を引数として取ることができる。

例 1:
```
{
    human(id: "1000") {
        name
        height
    }
}
```
```
{
    "data": {
        "human": {
            "name": "Luke Skywalker",
            "height": 1.72
        }
    }
}
```

例 2:
```
{
    human(id: "1000") {
        name
        height(unit: FOOT)
    }
}
```
```
{
    "data": {
        "human": {
            "name": "Luke Skywalker",
            "height": 5.6430448
        }
    }
}
```

## 操作タイプと名前 (Operation Type and Name)

ここまでの例は、`query` キーワードを省略したシンタックスを利用していた。
Query 以外の操作 (Mutation や Subscription) をする場合は、明示的に指定しなければならない。
`query`、`mutation`、`subscription` のいずれかの値をとることができる。

操作に名前をつけたい場合は、操作タイプの直後に書くことができる。

例
```
query HeroNameAndFriends {
    hero {
        name
        friends {
            name
        }
    }
}
```
```
{
    "data": {
        "hero": {
            "name": "R2-D2",
            "friends": [
                {
                    "name": "Luke Skywalker"
                },
                {
                    "name": "Han Solo"
                },
                {
                    "name": "Leia Organa"
                }
            ]
        }
    }
}
```

## エイリアス (Aliases)

返ってくるフィールドの名前を別の名前にすることができる。

例
```
query {
    empireHero: hero(episode: EMPIRE) {
        name
    }
    jediHero: hero(episode: JEDI) {
        name
    }
}
```
```
{
    "data": {
        "empireHero": {
            "name": "Luke Skywalker"
        },
        "jediHero": {
            "name": "R2-D2"
        }
    }
}
```

## 変数 (Variables)

これまでクエリ内に引数を直接書いてきたが、動的に変えたい場合がほとんどなはずである。
またクエリ文字列を直接操作するのは、様々な観点で好ましくない。

このような状況に対応するために、変数を渡すことができる。

例
```
query HeroNameAndFriends($episode: Episode) {
    hero(episode: $episode) {
        name
        friends {
            name
        }
    }
}
```
```
{
    "episode": "JEDI"
}
```
```
{
    "data": {
        "hero": {
            "name": "R2-D2",
            "friends": [
                {
                    "name": "Luke Skywalker"
                },
                {
                    "name": "Han Solo"
                },
                {
                    "name": "Leia Organa"
                }
            ]
        }
    }
}
```

## デフォルト変数 (Default Variables)

変数のデフォルトを設定することができる。

例
```
query HeroNameAndFriends($episode: Episode = JEDI) {
    hero(episode: $episode) {
        name
        friends {
            name
        }
    }
}
```

## ディレクティブ (Directives)

動的にクエリの構造を変更することもできる。
- `@include(if: Boolean)`: 引数が `true` の場合にのみ、このフィールドを含める。
- `@skip(if: Boolean)`: 引数が `true` の場合に、このフィールドをスキップする。

例
```
query Hero($episode: Episode, $withFriends: Boolean!) {
    hero(episode: $episode) {
        name
        friends @include(if: $withFriends) {
            name
        }
    }
}
```
```
{
    "episode": "JEDI",
    "withFriends": false
}
```
```
{
    "data": {
        "hero": {
            "name": "R2-D2"
        }
    }
}
```

## ページネーション (Pagination)

複数のオブジェクトの関係を表現する方法として最もシンプルな方法は、リストである。

例
```
query {
    hero {
        name
        friends {
            name
        }
    }
}
```
```
{
    "data": {
        "hero": {
            "name": "R2-D2",
            "friends": [
                {
                    "name": "Luke Skywalker"
                },
                {
                    "name": "Han Solo"
                },
                {
                    "name": "Leia Organa"
                }
            ]
        }
    }
}
```

リスト内の要素数が大きいため、クライアントが返して欲しい要素の数を指定したいような場合には、スライス (Slice) を利用することができる。

例
```
query {
    hero {
        name
        friends(first: 2) {
            name
        }
    }
}
```

上記の例では、リストの最初の 2 つ要素を取得していたが、追加でさらに 2 つの要素を取得したいといった場合に対応できるのが、ページネーション (Pagination) である。

ページネーションのシンプルな実装方法として、オフセットベースページネーション (Offset-based pagination) がある。
リスト内の最初の 2 つを飛ばした次の 2 つを要求する場合に、`friends(first: 2, offset: 2)` のようなクエリを使う方法である。
非常にシンプルではあるが、最初のクエリと次に続くクエリの間でデータ更新があった場合などに、同じ要素を返してしまう可能性や特定の要素が飛ばされてしまう場合があるという問題がある。

そこで、一般的なページネーションの実装として用いられるのが、カーソルベースページネーション (Cursor-based pagination) である。
具体的には `friends(first: 2, after: $friendId)` や `friends(first: 2, after: $friendCursor)` のように、仮にデータ更新が前のクエリと現在のクエリの間で発生しても取得し始める場所が一意に特定できるようなカーソルという識別子を渡す方法である。
このカーソルの情報を必ず返すようにし、そのカーソルを次のクエリの引数に含めることで、前のクエリの続きを取得することができる。

このカーソル情報を返り値に含めるために、多くの場合は単一の要素を示す `node` とそれを束ねる `edges` を使用し、`edges` の中にカーソルの情報も含める。

```
query {
    hero {
        name
        friends(first: 2) {
            edges {
                node {
                    name
                }
                cursor
            }
        }
    }
}
```

リストが全ての要素を返し切ったことを知る方法として、空のスライスが返されるまで繰り返す方法がある。
しかし、最後まで到達したことを知らせる情報があれば、空を確認するための追加のリクエストを減らすことができる。
これを実現するために、多くの場合、以下のようなクエリが利用される。

```
query {
    hero {
        name
        friends(first: 2) {
            totalCount
            edges {
                node {
                    name
                }
                cursor
            }
            pageInfo {
                endCursor
                hasNextPage
            }
        }
    }
}
```

以上のことを踏まえて、多くの場合、以下のような型システムを利用する。

```
interface Character {
    id: ID!
    name: String!
    friends: [Character]
    friendsConnection(first: Int, after: ID): FriendsConnection!
    appearsIn: [Episode]!
}

type FriendsConnection {
    totalCount: Int
    edges: [FriendsEdge]
    friends: [Character]
    pageInfo: PageInfo!
}

type FriendsEdge {
    cursor: ID!
    node: Character
}

type PageInfo {
    startCursor: ID
    endCursor: ID
    hasNextPage: Boolean!
}
```
```
hero {
    name
    friendsConnection(first: 2, after: "Y3Vyc29yMQ==") {
        totalCount
        edges {
            node {
                name
            }
            cursor
        }
        pageInfo {
            endCursor
            hasNextPage
        }
    }
}
```
```
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friendsConnection": {
        "totalCount": 3,
        "edges": [
          {
            "node": {
              "name": "Han Solo"
            },
            "cursor": "Y3Vyc29yMg=="
          },
          {
            "node": {
              "name": "Leia Organa"
            },
            "cursor": "Y3Vyc29yMw=="
          }
        ],
        "pageInfo": {
          "endCursor": "Y3Vyc29yMw==",
          "hasNextPage": false
        }
      }
    }
  }
}
```



# 参考リンク

- [GraphQL - Wikipedia](https://en.wikipedia.org/wiki/GraphQL)
- [Introduction to GraphQL | GraphQL](https://graphql.org/learn/)
- [GraphQL Specification Versions](https://spec.graphql.org/)
