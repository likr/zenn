---
title: "Reactチュートリアル2：レビューサイトを作ろう"
emoji: "🍜"
type: "tech"
topics: ["javascript", "react", "nodejs", "heroku"]
published: true
---

# 本資料について

本資料は日本大学文理学部情報科学科の開講科目「Web プログラミング」の教材として作成されました。本資料は下記のライセンスの範囲内で、当授業以外でも自由にご利用いただけます。

## 対象読者

本資料は、以下の教材を学習済み、もしくはそれと同等以上の知識を持っていることを前提としています。

- [React チュートリアル：犬画像ギャラリーを作ろう](https://zenn.dev/likr/articles/6be53ca64f29aa035f07)
- 基本情報技術者試験レベルの関係データベースの知識

## 本資料で学ぶこと

本資料では以下の内容を学びます。

- Express と Sequelize による API サーバー開発
- React と API サーバーの連携
- Cross-Origin Resourcer Sharing
- React によるルーティング
- Auth0 によるユーザー認証
- Heroku による API サーバーの公開

## ライセンス

[![license](https://i.creativecommons.org/l/by/4.0/88x31.png)](http://creativecommons.org/licenses/by/4.0/) この作品は[クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/)の下に提供されています。

## 準備

本チュートリアルを進めるにあたって、以下のツールをインストールしておいてください。

- [Git](https://git-scm.com/)
- [PostgreSQL](https://www.postgresql.org/)
- [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)

また、以下のサービスを利用するのでそれぞれのアカウントを取得しておいてください。

- [Netlify](https://www.netlify.com/)
- [Heroku](https://www.heroku.com/)
- [Auth0](https://auth0.com/)

# データベースを利用した Web アプリ

世の中の多くの Web アプリ（Web サイト・Web サービス等もひとまとめに Web アプリと呼ぶことにします）では、データベースを利用することでユーザーの情報を記録したり、ユーザー間で情報のやり取りを行っています。**関係データベース（RDB; Relational Database）** は、現在広く利用されているデータベースの一種であり、 オープンソースのデータベース管理システム（DBMS; Database Management System） として PostgreSQL や MySQL があります。

データベースを利用する Web アプリでは、ユーザーのブラウザ上で動くプログラムとアプリ提供者が所有するデータベースの間での通信が必要となります。このときユーザーのブラウザ上すなわちクライアント側で動くプログラムを **フロントエンド**と呼び、そうではないサーバー側で動くプログラムを**バックエンド** と呼びます。フロントエンドは主に HTML と CSS、JavaScript から構成され、ユーザーは Web サーバーからそれらのファイルを受け取ります。バックエンドはフロントエンドのプログラムから HTTP 等による通信を受け付け、リクエストの内容に応じてデータベース等と連携しながら処理をした結果をフロントエンドに返します。フロントエンドとバックエンドの間のデータのやり取りの形式には JSON を使用することが多いです。今回のチュートリアルでは、React によるフロントエンドだけではなく、データベースを利用したバックエンドの開発に取り組みます。

![Webアプリの構成](https://storage.googleapis.com/zenn-user-upload/tlb7w2v7tzp5ug2ew7tgdvr00j2a)

フロントエンドとバックエンドが連携するためには、その間の API を設計する必要があります。Web アプリにおける API の設計方針として主要なものの一つが **REST API**（もしくは RESTful API）です。REST API では、URI を持ったリソースとそれに対する操作に基づいてサーバーの処理を設計します。ユーザーや商品、コメントなど Web アプリで扱う様々な情報は REST API におけるリソースとして見做すことができます。リソースの URI とは、例えば ID が 42 のユーザーであれば、https://example.com/users/42 のように決めることができます。リソースに対する操作は HTTP のメソッドで表現されます。リソースの取得であれば GET、作成は POST や PUT、更新は PUT や PATCH、削除は DELETE で表されます。https://example.com/users/42 への GET リクエストは ID が 42 のユーザーの情報の取得と考えることができます。

ここからは早速、データベースと REST API を利用した Web アプリの開発に取り組んでいきましょう。本チュートリアルでは Node.js と PostgreSQL を利用した API サーバーを Heroku で公開します。また、ユーザーの認証には Auto0 を利用します。React で実装するフロントエンドは API サーバーと通信しながら画面の表示を行います。

# Web サービスの計画

本チュートリアルではラーメン店の口コミレビューを投稿できる Web アプリを開発します。ラーメン店の情報をたくさん集めるのが大変だったので著者の職場の近くのラーメン店のみを扱っています。アプリの名前は「日大文理ラーメンレビュー」とでもしておきましょう。データさえ用意すればラーメン店に限らず様々な飲食店や、飲食店以外の様々な商品のレビューサイトにも応用することができるでしょう。

下図はアプリの画面構成です。アプリは 3 つのページから構成されています。トップ画面ではレビューがたくさん付いている人気のラーメン店を表示します。実際のレビューアプリではもっと多くの情報を載せる必要があるかもしれません．ラーメン店一覧画面では、全てのラーメン店のリストを表示します。一度に全てのリストを表示してもユーザーは全部の情報に目を通さないかもしれないので、ページングを行うことで必要な情報に順次アクセスできるようにします。トップ画面とラーメン店一覧画面からラーメン店のリンクを押すとラーメン店詳細画面に移動します。この画面では、お店の地図やこのお店についた全てのレビューを表示できることに加えて、新しくレビューを書くことができます。レビューにはいつ誰が書いたのかといった情報が必要になるでしょう。 https://reverent-blackwell-e8f8e3.netlify.app/ で実際の動作の様子を確認することができます。

![アプリの構成](https://storage.googleapis.com/zenn-user-upload/g9da4g02e4fbj7o2eiw7txqhzwtp)

このアプリに必要なバックエンドの API を考えてみましょう。各ページの機能を整理していくと以下の表の機能が必要となります。REST API において、これらの機能ごとの URI を **エンドポイント** と呼びます。

| メソッド | パス                               | 機能                                                         |
| -------- | ---------------------------------- | ------------------------------------------------------------ |
| GET      | /restaurants                       | ラーメン店のリストを取得する                                 |
| GET      | /restaurants/:restaurantId         | ID が :restaurantId のラーメン店の情報を取得する             |
| GET      | /restaurants/:restaurantId/reviews | ID が :restaurantId のラーメン店のレビューのリストを取得する |
| POST     | /restaurants/:restaurantId/reviews | ID が :restaurantId のラーメン店のレビューを投稿する         |

`:restaurantId` はプレースホルダーであり、実際には `/restaurants/42` のようにリクエストをすると ID が 42 のラーメン店の情報を取得することになります。

各画面の機能と必要な API が決まったので実際に開発を進めていきましょう。このチュートリアルでは多くのコードを用意する必要があります。全てを打ち込むのは大変なのであらかじめいくつかのファイルを含んだリポジトリを用意しておきました。これをダウンロードして開発を始めましょう。

作業用ディレクトリの名前を `review-app` とします。`review-app` ディレクトリの中で以下のコマンドを実行しましょう。

```shell-session
$ git clone https://github.com/likr-lecture/react-tutorial2-client client
$ git clone https://github.com/likr-lecture/react-tutorial2-server server
```

クライアント用のファイルは `client` 、サーバー用のファイルは `server` に入れていきます。

# Node.js による API サーバー開発

初めにバックエンドの開発に取り組みます。バックエンドの API サーバーは、HTTP リクエストを受け取ってリクエスト内容に応じた処理を行ってレスポンスを返すプログラムとなります。
JavaScript は元々は Web ブラウザの上で動くように作られたプログラミング言語でしたが、Node.js の登場によって Web ブラウザだけではなくサーバーサイドのプログラムを JavaScript で書いて動かすことができるようになりました。本チュートリアルでは、API サーバーの実装に JavaScript を使用しますが、は JavaScript に限らず PHP や Ruby、Python など様々なプログラミング言語を利用することが可能です。

REST API では、どのリソースに対してどのような操作がリクエストされているかに応じて処理を振り分ける必要があります。その振り分け処理はどの API サーバー でもほとんど同じになるため、一般的にはそれを自分で実装することはなく、サーバーサイドの Web フレームワークを利用することになります。Node.js 用の人気のある Web フレームワークとして Express があります。ここでは Express を使って API サーバーを実装していきます。

API サーバーをはじめからデータベースや認証を使って完全に作るのではなく、まずは手始めに仮のデータを返すようにして最小限動作できるようにしてみましょう。仮のデータは `server/sample-data.js` に含まれています。API を決めるということは、そのインタフェースという境界が変わらなければ境界の向こうの処理が変わっても境界の手前には影響がないということです。そのため、まずは最小限の動作からはじめてインタフェースがうまく機能するか確かめていきましょう。

はじめに `server/package.json` を以下のように書き換えます。

```diff:server/package.json
 {
   "name": "server",
   "version": "1.0.0",
   "description": "",
   "main": "index.js",
+  "engines": {
+    "node": "18.x"
+  },
+  "type": "module",
   "scripts": {
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "start": "node index.js"
   },
   "keywords": [],
   "author": "",
   "license": "ISC"
 }
```

`"type": "module"` は Node.js で ES Modules 形式のインポート/エクスポートを行うために必要です。`engines` は後に Heroku で API サーバーを公開する際に動作させる Node.js のバージョンを指定します。できるだけローカルでの実行バージョンと、最終的に公開するプロダクション環境での実行バージョンは一致させておいたほうがよいでしょう。本チュートリアルでは、Node.js のバージョン 18 を前提に説明を行います。

次に以下のコマンドを実行して Express をインストールします。

```shell-session
$ npm i express
```

`server/index.js` にサーバープログラムを書いていきます。はじめに `express` と仮データをインポートし、Express アプリケーションのインスタンスを作成します。

```javascript
import express from "express";
import * as data from "./sample-data.js";

const app = express();
```

Express では `app.get(path, handler)` のように書くことで、`path` に対する GET リクエストを `handler` の関数で処理することができます。POST リクエストであれば、`app.post` 、PUT と DELETE もそれぞれ `app.put`と`app.delete` で同じように書くことができます。実際にエンドポイント `/restaurants` に対する処理を書くと以下のようになります。

```javascript
app.get("/restaurants", async (req, res) => {
  const limit = +req.query.limit || 5;
  const offset = +req.query.offset || 0;
  const restaurants = data.restaurants;
  res.json({
    rows: restaurants.slice(offset, offset + limit),
    count: data.restaurants.length,
  });
});
```

`handler` の関数の第 1 引数にはリクエストの情報を含んだ `Request` オブジェクトが、第 2 引数にはレスポンスを返すための `Response` オブジェクトが渡されます。

`/restaurants` エンドポイントでは、ページングを行うために取得件数を表す `limit` とリストの何番目から取得を行うかを表す `offset` をクエリ文字列として受け取っています。クエリ文字列とは URL に含まれる `?` より後ろの部分で、具体的には `/restaurants?limit=3&offset=5` のようなリクエストだとリストの先頭 5 番目から 3 件のデータを返します。`limit` と `offset` は省略することができ、その場合はそれぞれ 5 と 0 をデフォルト値としています。クエリ文字列に含まれるパラメータは `req.query` から取り出すことができます。

仮データのラーメン店情報は `data.restaurants` に含まれており、今は単に配列の `slice` メソッドによって必要なレコードを取り出します。レスポンスは `rows` と `count` の 2 つのプロパティをもったオブジェクトで、`rows` はラーメン店の情報の配列を、`count` はラーメン店の全件数を表しています。レスポンスを JSON 形式で返すためには `res.json` にオブジェクトを渡します。

続いてエンドポイント `/restaurants/:restaurantId` に対する処理です。

```javascript
app.get("/restaurants/:restaurantId", async (req, res) => {
  const restaurantId = +req.params.restaurantId;
  const restaurant = data.restaurants.find(
    (restaurant) => restaurant.id === restaurantId,
  );
  if (!restaurant) {
    res.status(404).send("not found");
    return;
  }
  res.json(restaurant);
});
```

このエンドポイントでは `:restaurantId` がプレースホルダーとなっていて、実際には `/restaurants/1` や `/restaurants/42` のように具体的な ID が渡されます。プレースホルダーのパラメータを取り出すには `req.params` を使います。

取り出した `restaurantId` と ID が一致するラーメン店を `data.restaurants` から探しますが、もし見つからなかったらリクエストされたリソースが存在しないことをクライアントに知らせなければいけません。このようなときには HTTP のステータスコード 404 でレスポンスを返すと良いでしょう。`res.status(code)` でレスポンスのステータスコードを設定することができます。ここでは、続けて `send(message)` を呼び出すことでプレーンテキストでレスポンスを返しています。該当するラーメン店が見つかった場合はそのオブジェクトを`res.json` で返しています。

もう一つ `/restaurants/:restaurantId/reviews` のエンドポイントを以下のように実装します。上 2 つのエンドポイントで行っている処理の組み合わせなので内容は理解できるでしょうか？

```javascript
app.get("/restaurants/:restaurantId/reviews", async (req, res) => {
  const restaurantId = +req.params.restaurantId;
  const limit = +req.query.limit || 5;
  const offset = +req.query.offset || 0;
  const restaurant = data.restaurants.find(
    (restaurant) => restaurant.id === restaurantId,
  );
  if (!restaurant) {
    res.status(404).send("not found");
    return;
  }
  const reviews = data.reviews.filter(
    (review) => review.restaurantId === restaurantId,
  );
  res.json({
    count: reviews.length,
    rows: reviews.slice(offset, offset + limit),
  });
});
```

エンドポイントの実装が終わったら以下のようにサーバーの起動を行います。

```javascript
const port = process.env.PORT || 5000;
app.listen(port, () => {
  console.log(`Listening at http://localhost:${port}`);
});
```

`app.listen(port, handler)` は `port` のポート番号でサーバーを起動し、起動が終わった時の処理を `handler` で行うことができます。
ローカル開発でのポート番号は 5000 番を使っていますが、後で Heroku で API サーバーを公開する際にはポート番号が環境変数で渡されるため、環境変数に `PORT` が設定されていたらそのポート番号を使うようにしています。。

`server/index.js` の全体を以下に示します。

```javascript:server/index.js
import express from "express";
import * as data from "./sample-data.js";

const app = express();

app.get("/restaurants", async (req, res) => {
  const limit = +req.query.limit || 5;
  const offset = +req.query.offset || 0;
  const restaurants = data.restaurants;
  res.json({
    rows: restaurants.slice(offset, offset + limit),
    count: data.restaurants.length,
  });
});

app.get("/restaurants/:restaurantId", async (req, res) => {
  const restaurantId = +req.params.restaurantId;
  const restaurant = data.restaurants.find(
    (restaurant) => restaurant.id === restaurantId
  );
  if (!restaurant) {
    res.status(404).send("not found");
    return;
  }
  res.json(restaurant);
});

app.get("/restaurants/:restaurantId/reviews", async (req, res) => {
  const restaurantId = +req.params.restaurantId;
  const limit = +req.query.limit || 5;
  const offset = +req.query.offset || 0;
  const restaurant = data.restaurants.find(
    (restaurant) => restaurant.id === restaurantId
  );
  if (!restaurant) {
    res.status(404).send("not found");
    return;
  }
  const reviews = data.reviews.filter(
    (review) => review.restaurantId === restaurantId
  );
  res.json({
    count: reviews.length,
    rows: reviews.slice(offset, offset + limit),
  });
});

const port = process.env.PORT || 5000;
app.listen(port, () => {
  console.log(`Listening at http://localhost:${port}`);
});
```

`server` ディレクトリ内で以下のコマンドを実行してサーバーを起動しましょう。

```shell-session
$ npm start
```

クライアントはまだできていませんが、Web ブラウザを使ってサーバーの動作を確認することができます。試しにブラウザで http://localhost:5000/restaurants にアクセスしてみましょう。以下のようにレスポンスの JSON 文字列がブラウザに表示されていれば成功です。開発者ツールを使えば、それを JSON として解釈した結果も確認することができます。

![サーバーからのレスポンス](https://storage.googleapis.com/zenn-user-upload/du6miv7nqcjggqdmxqydq7zayivk)

クエリ文字列がうまく動作しているか確認するために http://localhost:5000/restaurants?limit=3&offset=5 なども試してみましょう。`server/sample-data.js` の中身も読んでみて意図した通りの結果が返ってきているか確認しましょう。また、他のエンドポイントの http://localhost:5000/restaurants/1 や http://localhost:5000/restaurants/1/reviews にもアクセスしてみましょう。これらは正しくデータが返ってくるはずですが、存在しない ID の http://localhost:5000/restaurants/42 だと異なる結果となるはずです。

# React Router を用いた複数ページ Web アプリの開発

API サーバーが仮データを正しく返してくれることが確認できました。次に、API サーバーからデータを受け取って Web ページを表示するフロントエンドの開発に移りましょう。基本的には[前回](https://zenn.dev/likr/articles/6be53ca64f29aa035f07)と同様ですが、今回の Web ページは複数のページから構成されておりページの切り替えが必要な点が異なります。React で複数ページから構成される Web アプリを開発するためには React Router を使用します。

まずは `client` ディレクトリ内で以下のコマンドを実行して必要なパッケージのインストールを行います。

```shell-session
$ npm i react react-dom react-router-dom bulma
$ npm i -D vite @vitejs/plugin-react
```

React Router を利用するために `react-router-dom` をインストールしています。

`client/src/App.jsx` を以下の内容で作成しましょう。

```jsx:client/src/App.jsx
import { BrowserRouter, Route, Routes } from "react-router-dom";
import { RootPage } from "./pages/Root.jsx";
import { RestaurantDetailPage } from "./pages/RestaurantDetail.jsx";
import { RestaurantListPage } from "./pages/RestaurantList.jsx";

function Header() {
  return (
    <section className="hero is-warning">
      <div className="hero-body">
        <div className="container">
          <h1 className="title">
            日大文理
            <br className="is-hidden-tablet" />
            ラーメンレビュー
          </h1>
        </div>
      </div>
    </section>
  );
}

function Footer() {
  return (
    <footer className="footer ">
      <div className="content">
        <p className="has-text-centered">
          これは日本大学文理学部情報科学科の開講科目「Web
          プログラミング」の教材として作成されたサンプルアプリケーションです。
        </p>
      </div>
    </footer>
  );
}

export function App() {
  return (
    <BrowserRouter>
      <Header />
      <section className="section has-background-warning-light">
        <div className="container">
          <div className="block has-text-right">
            <button className="button is-warning is-inverted is-outlined">
              ログイン
            </button>
          </div>
          <Switch>
            <Route path="/" element={<RootPage />} />
            <Route path="/restaurants" element={<RestaurantListPage />} />
            <Route
              path="/restaurants/:restaurantId"
              element={<RestaurantDetailPage />}
            />
          </Switch>
        </div>
      </section>
      <Footer />
    </BrowserRouter>
  );
}
```

`App` コンポーネントに注目してください。React Router を使って、ブラウザが表示している URL に応じて React で表示させるコンポーネントを切り替えるように設定しています。ここでは、`react-router-dom` からインポートした `BrowserRouter` と `Route` 、 `Routes` という 3 つのコンポーネントが登場します。`BrowserRouter` コンポーネントは、React Router が管理するコンポーネントの範囲を設定します。基本的にはアプリケーションのコンポーネント全体を `BrowserRouter` コンポーネントの子要素にしておくとよいでしょう。`Routes` コンポーネントは、URL によって切り替わる要素の場所を設定します。`Route` コンポーネントは、`path` 属性を持ち、URL が `path` と一致したときにページに表示させる内容を `element` props で設定します。

このアプリは 3 つの画面が存在するため、それぞれに URL を決めて 3 つのルートを設定しています。`/` はトップ画面、`/restaurants` はラーメン店一覧画面、`/restaurants/:restaurantId` はラーメン店詳細画面にそれぞれ対応しています。それぞれのルートに対応する具体的な表示内容は `Route` コンポーネントの子要素に持たせます。ここでは、URL が `/` のとき `RootPage` コンポーネント、`/restaurants` のとき `RestaurantListPage` コンポーネント、`/restaurants/:restaurantId` のとき `RestaurantDetailPage` コンポーネントがレンダリングされます。なお、Express と同様に `:restaurantId` はプレースホルダーになっていて、具体的な ID と置き換えられます。

次に `client/src/pages/Root.jsx` を以下の内容で作成します。

```jsx:client/src/pages/Root.jsx
import { useEffect, useState } from "react";
import { Link } from "react-router-dom";
import { getRestaurants } from "../api.js";
import { Loading, Restaurant } from "../components";

export function RootPage() {
  const [restaurants, setRestaurants] = useState(null);

  useEffect(() => {
    getRestaurants({ limit: 3 }).then((data) => {
      setRestaurants(data);
    });
  }, []);

  return (
    <>
      <h2 className="title is-3">人気のラーメン店</h2>
      <div className="block">
        {restaurants == null ? (
          <Loading />
        ) : (
          restaurants.rows.map((restaurant) => {
            return <Restaurant key={restaurant.id} restaurant={restaurant} />;
          })
        )}
      </div>
      <div className="has-text-right">
        <Link className="button is-warning" to="/restaurants">
          全てのラーメン店を見る
        </Link>
      </div>
    </>
  );
}
```

ほとんどは前回のチュートリアルと同じですが、新しく React Router の `Link` コンポーネントが登場しています。`Link` コンポーネントは、React Router を使った React アプリケーションでのページ遷移に使用するコンポーネントで HTML の `a` 要素と同じような役割をします。`Link` コンポーネントの `to` 属性に遷移先のパスを指定することで、ユーザーがその要素をクリックしたときに画面の遷移を行います。

ここ以外にも最初から含まれているプログラムの中にも何ヶ所か `Link` コンポーネントを使用しているところがあるので、探してどのような使い方をしているか確認すると良いでしょう。

# フロントエンドからの API リクエスト

ここまでで全てのコンポーネントの実装が終わりました。最後に API サーバーへのリクエストを行う関数を実装していきましょう。

開発環境では、現在 http://localhost:5000 で動いている開発用の API サーバーにアクセスしていますが、最終的には Heroku 上で公開をするため、API サーバーの URL を切り替える必要が出てきます。このように、開発環境と本番環境（最終的に公開する環境）でパラメータを切り替える必要がある場合には環境変数を利用すると良いでしょう。`vite` を使っていれば、開発環境用の環境変数を `.env.development` 、本番環境用の環境変数を `.env.production` で設定することができます。これらのファイルには `VITE_` から始まる環境変数名を記述します。

API サーバーの URL、正確にはプロトコルとホスト名、ポート番号の 3 つがセットになった **オリジン** を `VITE_API_ORIGIN` という名前で設定しておきます。`client/.env.development` を以下の内容で作成しましょう。

```sh:client/.env.development
VITE_API_ORIGIN=http://localhost:5000
```

フロントエンドからの API リクエストには `fetch` を使うことができましたが、API サーバーのエンドポイントに対応した関数を作っておくとコンポーネントの実装が楽になるでしょう。`client/src/api.js` を以下のように実装します。

```javascript:client/src/api.js
async function request(path, options = {}) {
  const url = `${import.meta.env.VITE_API_ORIGIN}${path}`;
  const response = await fetch(url, options);
  return response.json();
}

export async function getRestaurants(arg = {}) {
  const params = new URLSearchParams(arg);
  return request(`/restaurants?${params.toString()}`);
}

export async function getRestaurant(restaurantId) {
  return request(`/restaurants/${restaurantId}`);
}

export async function getRestaurantReviews(restaurantId, arg = {}) {
  const params = new URLSearchParams(arg);
  return request(`/restaurants/${restaurantId}/reviews?${params.toString()}`);
}
```

`/restaurants` と `/restaurants/:restaurantId` 、 `/restaurants/:restaurantId/reviews` に対して GET リクエストを行う関数をそれぞれ `getRestaurants` 、 `getRestaurant` 、`getRestaurantReviews` としています。これらの間の共通の処理は `request` 関数にまとめています。

ここまででフロントエンドを動作させる一通りのプログラムを書き終えたはずなので、 `client` ディレクトリ内で `npm start` を実行することでフロントエンドの開発サーバーを起動し、http://localhost:5173 にアクセスしてみましょう。ラーメン店の情報が表示されて欲しいところですが、いつまで待っても「loading...」の表示が消えません。開発者ツールのコンソールを確認すると以下のようなエラーが見られるでしょう。

```
Access to fetch at 'http://localhost:5000/restaurants?limit=3' from origin 'http://localhost:5173' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
```

現在の Web では、セキュリティ上の理由で異なるオリジンへの API リクエストは制限されています。フロントエンドのオリジンが http://localhost:5173 であるのに対して API サーバーのオリジンが http://localhost:5000 と異なるため、この制限に引っかかっているということです。これを解消するためには API サーバー側でどのオリジンからのアクセスを許可するという設定をしなければいけません。どこのオリジンからアクセスされても良いような API サーバーであったとしても、「どこのオリジンからもアクセスされても良い」ということを明示的に設定しておく必要があります。

フロントエンドと異なるオリジンとの通信は **CORS（Cross-Origin Resource Sharing）** と呼ばれます。改めてサーバー側プログラムに戻って CORS の設定を追加しましょう。

まずは `server` ディレクトリ内で以下のコマンドを実行して `cors` パッケージをインストールします。

```shell-session
$ npm i cors
```

次に `server/index.js` を以下のように編集することで CORS の設定が可能となります。ここでは、この API サーバーはどのオリジンからのリクエストも受け付けるようになっています。

```diff:server/index.js
 import express from "express";
+import cors from "cors";
 import * as data from "./sample-data.js";

 const app = express();
+app.use(cors());
```

API サーバーを再起動した後にもう一度 http://localhost:5173 にアクセスしてみましょう。こんどは API サーバーからのレスポンスを受け取って正しくページの表示ができているはずです。ラーメン店の名前や「全てのラーメン店を見る」のボタンを押してページの遷移がうまくいくかも確認してみましょう。

# OR マッパーによるデータベースの利用

API サーバーとフロントエンドが連携することで実際の Web サイトらしい動きが実現できましたが、API サーバーが返しているのは仮データのままでした。次はデータベースを導入して、実際のデータをやり取りできるようにしていきましょう。

ここからは、ローカルの開発環境からアクセスできる PostgreSQL のサーバーを用意しておく必要があります。データベースのホストを `localhost` 、ポート番号を `5432` 、ユーザー名を `postgres` 、パスワードを `postgres` 、データベース名を `review_app` として説明をします。異なる設定にしている場合はプログラムを適宜読み替えてください。データベースサーバーは好きな方法で用意してもらって構いませんが、例えば Docker であれば以下のコマンドで PostgreSQL のサーバーを起動することが可能です。

```shell-session
$ docker run -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=review_app -p 5432:5432 postgres:12
```

関係データベースでは一般的にクエリ言語の SQL を通じて操作を行います。基本的には API サーバーのプログラムが SQL を組み立てて、それをデータベースサーバーとやり取りすることになります。SQL で取り出した情報は JavaScript 等のプログラミング言語で扱いやすい状態になっていると便利でしょう。O/R マッパーは、SQL の組み立てやクエリ結果のオブジェクトへの変換を行うライブラリで、Node.js では Sequelize という有名な O/R マッパーがあります。

`server` ディレクトリで以下のコマンドを実行して必要なパッケージをインストールします。

```shell-session
$ npm i sequelize pg pg-hstore
```

Sequelize を使う上で、はじめにデータベースのテーブルとプログラムの関係を定義します。この定義情報を `server/models.js` に書いていきましょう。

はじめに、ライブラリのインポートとデータベースの接続を行います。

```javascript
import Sequelize from "sequelize";

const { DataTypes } = Sequelize;

const url =
  process.env.DATABASE_URL ||
  "postgres://postgres:postgres@localhost:5432/review_app";
export const sequelize = new Sequelize(url);
```

`new Sequelize(url)` は引数の URL に対してデータベースの接続を行います。PostgreSQL へ接続するための URL は `postgres://[ユーザー名]:[パスワード]@[ホスト名]:[ポート番号]/[データベース名]` の形式になります。資料と異なる設定の場合は `"postgres://postgres:postgres@localhost:5432/review_app"` の部分を変更してください。Heroku の環境ではこの URL が環境変数で与えられるので、環境変数が設定されていればそちらを優先して読み込むようにしています。

次に、テーブルの情報を定義していきます。まずはレビューを投稿したユーザーの情報を表す `users` テーブルを定義します。

```javascript
export const User = sequelize.define(
  "user",
  {
    sub: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    nickname: {
      type: DataTypes.STRING,
      allowNull: false,
    },
  },
  { underscored: true },
);
```

テーブルの定義は `sequelize.define` で行います。第 1 引数がテーブル名、第 2 引数が列の情報、第 3 引数がその他のオプションとなります。列の情報はオブジェクトで指定し、オブジェクトのプロパティ名が列名、その列の情報が値となります。列の情報の値にはデータ型を表す `type` が必要です。`DataTypes` を使用して列のデータ型を定義します。データ型には、文字列を表す `STRING` や 整数を表す `INTEGER` 、長い文字列を表す `TEXT` などがあります。列の情報に `allowNull: false` を指定するとその列に対して NOT NULL 制約が設定されます。

同様に `restaurants` テーブルと `reviews` テーブルを定義します。

```javascript
export const Restaurant = sequelize.define(
  "restaurant",
  {
    name: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    image: {
      type: DataTypes.STRING,
    },
    map: {
      type: DataTypes.TEXT,
    },
  },
  { underscored: true },
);

export const Review = sequelize.define(
  "review",
  {
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      references: {
        model: User,
      },
    },
    restaurantId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      references: {
        model: Restaurant,
      },
    },
    title: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    comment: {
      type: DataTypes.STRING,
      allowNull: false,
    },
  },
  { underscored: true },
);
```

`reviews` テーブルと `restaurants` テーブル、`reviews` テーブルと `users` テーブルはどちらも 1 対多の関係にあります。列の情報に `reference` を与えると外部キー制約が設定されます。

最後にテーブル間の関係を定義します。これにより、関連するテーブルの JOIN が可能になります。

```javascript
Restaurant.hasMany(Review);
Review.belongsTo(Restaurant);
User.hasMany(Review);
Review.belongsTo(User);
```

`server/models.js` の全体は以下のようになります。

```javascript:server/models.js
import Sequelize from "sequelize";

const { DataTypes } = Sequelize;

const url =
  process.env.DATABASE_URL ||
  "postgres://postgres:postgres@localhost:5432/review_app";
export const sequelize = new Sequelize(url);

export const User = sequelize.define(
  "user",
  {
    sub: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    nickname: {
      type: DataTypes.STRING,
      allowNull: false,
    },
  },
  { underscored: true },
);

export const Restaurant = sequelize.define(
  "restaurant",
  {
    name: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    image: {
      type: DataTypes.STRING,
    },
    map: {
      type: DataTypes.TEXT,
    },
  },
  { underscored: true },
);

export const Review = sequelize.define(
  "review",
  {
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      references: {
        model: User,
      },
    },
    restaurantId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      references: {
        model: Restaurant,
      },
    },
    title: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    comment: {
      type: DataTypes.STRING,
      allowNull: false,
    },
  },
  { underscored: true },
);

Restaurant.hasMany(Review);
Review.belongsTo(Restaurant);
User.hasMany(Review);
Review.belongsTo(User);
```

次にデータベースの初期化と、必要なデータをデータベースに追加するプログラムを作成します。まだレビューを投稿する機能がないので、ユーザーとレビューの情報は仮データの内容をデータベースに投稿することにします。`server/migration.js` を以下の内容で作成してください。

```javascript:server/migration.js
import { sequelize, Restaurant, Review, User } from "./models.js";
import * as data from "./sample-data.js";

await sequelize.sync({ force: true });
for (const { name, image, map } of data.restaurants) {
  await Restaurant.create({ name, image, map });
}
for (const { sub, nickname } of data.users) {
  await User.create({ sub, nickname });
}
for (const { title, comment, userId, restaurantId } of data.reviews) {
  await Review.create({ title, comment, userId, restaurantId });
}
```

`server` ディレクトリ内で以下のコマンドを実行しましょう。エラーが起きたら、データベースの接続情報が正しいか確認しましょう。

```shell-session
$ node migration.js
```

`server/index.js` を、データベースからデータを読み込むように以下のように修正しましょう。

```diff:server/index.js
 import express from "express";
-import * as data from "./sample-data.js";
+import sequelize from "sequelize";
+import { Restaurant, Review, User } from "./models.js";

 const app = express();

 app.get("/restaurants", async (req, res) => {
   const limit = +req.query.limit || 5;
   const offset = +req.query.offset || 0;
-  const restaurants = data.restaurants;
-  res.json({
-    rows: restaurants.slice(offset, offset + limit),
-    count: data.restaurants.length,
-  });
+  const restaurants = await Restaurant.findAndCountAll({
+    attributes: {
+      include: [
+        [
+          sequelize.literal(
+            `(SELECT COUNT(*) FROM reviews AS r WHERE r.restaurant_id = restaurant.id)`,
+          ),
+          "review_count",
+        ],
+      ],
+    },
+    include: { model: Review, limit: 3, include: { model: User } },
+    order: [[sequelize.literal("review_count"), "DESC"]],
+    limit,
+    offset,
+  });
+  res.json(restaurants);
 });

 app.get("/restaurants/:restaurantId", async (req, res) => {
   const restaurantId = +req.params.restaurantId;
-  const restaurant = data.restaurants.find(
-    (restaurant) => restaurant.id === restaurantId
-  );
+  const restaurant = await Restaurant.findByPk(restaurantId);
   if (!restaurant) {
     res.status(404).send("not found");
     return;
   }
   res.json(restaurant);
 });

 app.get("/restaurants/:restaurantId/reviews", async (req, res) => {
   const restaurantId = +req.params.restaurantId;
   const limit = +req.query.limit || 5;
   const offset = +req.query.offset || 0;
-  const restaurant = data.restaurants.find(
-    (restaurant) => restaurant.id === restaurantId
-  );
+  const restaurant = await Restaurant.findByPk(restaurantId);
   if (!restaurant) {
     res.status(404).send("not found");
     return;
   }
-  const reviews = data.reviews.filter(
-    (review) => review.restaurantId === restaurantId
-  );
-  res.json({
-    count: reviews.length,
-    rows: reviews.slice(offset, offset + limit),
-  });
+  const reviews = await Review.findAndCountAll({
+    include: { model: User },
+    where: { restaurantId },
+    limit,
+    offset,
+  });
+  res.json(reviews);
 });

 const port = process.env.PORT || 5000;
 app.listen(port, () => {
   console.log(`Listening at http://localhost:${port}`);
 });
```

ここで使用している Sequelize の主要な機能を紹介します。

エンドポイント `/restaurants/:restaurantId` の処理では、`Restaurant.findByPk(restaurantId)` を呼び出しています。これは、`restaurants` テーブルから `id` が `restaurantId` と一致するレコードを取得しています。`id` が 1 のレコードを取得する際には `SELECT * FROM restaurants WHERE id = 1;` のような SQL が発行されます。該当するレコードが存在しなければ `null` が返されるため、変更前と同じように 戻り値が `null` であればステータスコード 404 の処理を行うことができます。

次にエンドポイント `/restaurants/:restaurantId/reviews` の処理を見てみましょう。ここでは、`reviews` テーブルの `restaurantId` がリクエストされた `restaurantId` と一致するレコードを `offset` と `limit` で指定された分だけ取得します。SELECT 文は `Review.findAll` によって行いますが、 `Review.findAndCountAll` はレコードの取得と同時に、条件に該当する総レコード数を `offset` と `limit` を無視して取得します。これはページングを行うときに便利になります。SQL の SELECT 文には、WHERE や ORDER BY などのいくつかの句が登場しました。それらの句は`findAll` および `findAllCountAll` の引数で指定することができます。ここでは、`where: { restaurantId }` で `restaurantId` による絞り込みを行い、該当するレコードの先頭から `offset` 番目から `limit` 件までを取得しています。`include` は関連が設定されたテーブルを JOIN した上で結果を取得するように指定します。ここでは `users` テーブルを JOIN することで、レビューを書いたユーザーの情報を含んだ上でレスポンスを返しています。

最後にエンドポイント `/restaurants` のクエリはやや複雑です。このクエリは、ラーメン店の情報を付いているレビュー件数の降順でソートし、なおかつ 1 つのラーメン店につき 3 件までのレビューを含んだ状態でレコードを取得しています。これは Sequelize の内部では 2 つの SQL 文に分けて発行されます。

`findAndCountAll` の戻り値の形式は仮データを使っていた時の形式と同じです（というよりは、仮データを使ったプログラムは Sequelize に合わせるように作りました）。そのため、`rows` には取得したレコードが、`count`には総件数が含まれています。

ここまでプログラムを修正したらサーバーを再起動して http://localhost:5173 からアプリにアクセスしてみましょう。表示される内容は変わらないはずですが、正しく表示されていたら成功です。

# Heroku による API サーバーの公開

まだレビューの投稿機能が作れていませんが、ここで一旦アプリの公開を行っておきましょう。API サーバーは Heroku を、フロントエンドは Netlify を使って公開します。

まずはサーバーの公開を行います。Heroku に API サーバーを実行する方法を知らせるために、`server/Procfile` を以下の内容で作成します。`node index.js` はサーバーを実行するコマンドを表しています。

```sh:server/Procfile
web: node index.js
```

次に、`server` ディレクトリ内で以下のコマンドを実行し、Heroku への API サーバーのデプロイを行います。

```shell-session
$ git add .
$ git commit -m 'first version'
$ heroku create
$ git push heroku master
$ heroku ps:scale web=1
```

続けて、以下のコマンドを実行して Heroku 上での PostgreSQL サーバーを有効化します。

```shell-session
$ heroku addons:create heroku-postgresql:hobby-dev
```

今はデータベースが作成された直後の空状態です。Heroku ではローカルのデータベースサーバーの内容を Heroku のデータベースサーバーに全てコピーすることができます。以下のコマンドを実行しましょう。`postgres://postgres:postgres@localhost:5432/review_app` はローカルデータベースの接続情報を表しているので、自分の環境に応じて変更してください。

```shell-session
$ heroku pg:push postgres://postgres:postgres@localhost:5432/review_app DATABASE_URL
```

ここまでできたら、以下のコマンドでブラウザから API サーバーにアクセスしてみましょう。

```shell-session
$ heroku open
```

最初は`/` にアクセスしているので「Cannot GET /」と表示されるでしょう。URL に `/restaurants` を加えてみましょう。API が正しくレスポンスを返していたら成功です。

もし API サーバーがうまく動作しなければプログラムやデータベースの設定を見直しましょう。サーバーのプログラムを修正した場合は、以下のコマンドで修正をコミットして Heroku のリポジトリに push します。詳細は Git の使い方を調べてみましょう。

```shell-session
$ git add .
$ git commit -m 'update'
$ git push heroku master
```

データベースの同期をやり直すには以下のコマンドを実行します。

```shell-session
$ node migration.js
$ heroku pg:reset
$ heroku pg:push postgres://postgres:postgres@localhost:5432/review_app DATABASE_URL
```

# Netlify によるフロントエンドの公開

続けてフロントエンドを Netlify で公開します。いくつかの設定ファイルを追加しましょう。

API サーバーを公開したことで、本番環境での API サーバーの URL が決まりました。`client/.env.production` を作成して、API サーバーの URL を記載しましょう。Heroku で公開した API サーバーにはランダムなホスト名が割り振られます。本資料では https://desolate-lowlands-46852.herokuapp.com を使用しますが、自分の API の URL を使用してください。

```sh:client/.env.production
VITE_API_ORIGIN=https://desolate-lowlands-46852.herokuapp.com
```

次に、本番環境で React Router を動作させるための設定を加えます。ラーメン店一覧画面のパスは `/restaurants` となっていますが、何も設定していなければ Netlify の Web サーバー上には `/restaurants` という URL のリソースはなく、URL 欄に直接入力した場合や、ブラウザのリロードでページを更新したときに 404 のページが表示されます。これを避けるために、404 のページを表示する変わりに `index.html` を表示させるような設定が必要です。

Netlify では、公開ディレクトリに `_redirects` ファイルを作成して転送情報を書くことで実現可能です。`cilent/public/_redirects` を以下の内容で作成しましょう。

```:client/public/_redirects
/*    /index.html   200
```

前回はファイルを手動でアップロードすることで Netlify へのデプロイを行いましたが、頻繁にページを更新する場合に手作業が生じるのは手間になります。今回は `netlify-cli` を使ってコマンドから Netlify へのデプロイを行いましょう。

`client` ディレクトリ内で以下のコマンドを実行して `netlify-cli` をインストールしましょう。

```shell-session
$ npm i -D netlify-cli
```

`package.json` を編集してデプロイ用のショートカットコマンドを用意しておきましょう。

```diff:server/client.json
 {
   "name": "client",
   "version": "1.0.0",
   "description": "",
   "main": "index.js",
   "scripts": {
     "build": "react-scripts build",
+    "deploy": "npm run build && netlify deploy --prod -d build",
     "start": "node index.js"
   },
   "keywords": [],
   "author": "",
   "license": "ISC"
 }
```

以下のコマンドを実行すると Netlify へのデプロイが行われます。

```shell-session
$ npm run deploy
```

実行結果に表示される URL にアクセスしてアプリが動作しているか確認しましょう。以降の説明で、Netlify にデプロイされた URL が必要となります。資料中では https://reverent-blackwell-e8f8e3.netlify.app/ を使用しますが、自分の URL と読み替えてください。

# ユーザー認証

最後にユーザーからのレビューの投稿機能を開発していきましょう。レビューの投稿にはユーザーにログインをしてもらって、誰がどのレビューを投稿したのかを記録する必要があります。ユーザーがログインできるようにするということは、メールアドレスやパスワード等のユーザーの個人情報を管理しなければならないことを意味します。ユーザーログインの仕組みを、不正アクセスや情報漏洩がないように自分で設計して実装するのはなかなか難しいことです。そこで、本チュートリアルではユーザーの認証機能を提供してくれる Auth0 というサービスを利用します。Auth0 を利用することで少ない手間で安全なユーザー認証の仕組みをアプリに組み込むことができます。

Auth0 での管理単位はテナントと呼ばれます。Auth0 に登録すると開発用のテナントが割り当てられているでしょう。それを利用しても構いませんし、新しくテナントを作成することもできます。

メニューの「Applications」を開き、「CREATE APPLICATION」と書かれたボタンを押して新しくアプリケーションを作成しましょう。開発環境と本番環境で 2 つのアプリケーションを作る必要があります。「Name」は何でも構いませんが、資料流では開発環境は「Development」、本番環境は「Review App」としておきます。「Choose an application type」では、「Single Page Web Applications」を選び「CREATE」ボタンを押してください。

![](https://storage.googleapis.com/zenn-user-upload/9dpl7o26r8krnr9zsbepsajmc94j)

作成したアプリケーションの画面で表示されている「Client ID」をプログラム中に記載する必要があるので、このページの表示の仕方を覚えておきましょう。

![](https://storage.googleapis.com/zenn-user-upload/91n00ogeseb15pugtu6gevwweqbt)

また、それぞれのアプリケーションで「Allowd Callback URLs」と「Allowed Logout URLs」、「Allowed Web Origins」、「Allowed Origins（CORS）」に開発環境と本番環境のフロントエンドの URL を入力し、画面下部の「SAVE CHANGES」ボタンを押して保存します。開発環境の URL は http://localhost:5173 、本番環境の URL は https://reverent-blackwell-e8f8e3.netlify.app/ のような Netlify にデプロイした結果の URL となります。

![](https://storage.googleapis.com/zenn-user-upload/92zxvte7p45qson55e9470qqz32z)

次にメニュー「APIs」を開き、「CREATE API」から API を登録します。これは、API サーバーに認証機能を加えるもので Name は例えば「Review App API」、Identifier は Heroku で公開された API サーバーの URL としてください。

![](https://storage.googleapis.com/zenn-user-upload/sgdfraqkr5tcv8wa03watuawkxyo)

以上で Auth0 のサイト上での設定は完了です。これらをプログラムに反映させていきましょう。

## サーバーの実装

サーバー側で認証機能を加えるために、 `express-jwt`と `jwks-rsa` パッケージをインストールします。また、Auth0 のサーバーからユーザー情報を取得するために、Node.js で Fetch API を利用する`node-fetch` をインストールします。`server` ディレクトリで以下のコマンドを実行しましょう。

```shell-session
$ npm i node-fetch express-jwt jwks-rsa
```

`server/auth0.js` を作成して、認証処理とユーザー情報取得処理を実装します。このあたりは Auth0 のドキュメントに載っている実装例なので、細かい中身までは理解できなくても問題ないでしょう。 **12 行目の `audience` を自分の API サーバーのオリジンに設定するのを忘れないでください。**

```javascript:server/auth0.js
import fetch from "node-fetch";
import jwt from "express-jwt";
import jwksRsa from "jwks-rsa";

export const checkJwt = jwt({
  secret: jwksRsa.expressJwtSecret({
    cache: true,
    rateLimit: true,
    jwksRequestsPerMinute: 5,
    jwksUri: `https://dev-ajrt-kp3.us.auth0.com/.well-known/jwks.json`,
  }),
  audience: "https://desolate-lowlands-46852.herokuapp.com",
  issuer: `https://dev-ajrt-kp3.us.auth0.com/`,
  algorithms: ["RS256"],
});

export async function getUser(token) {
  const auth0Request = await fetch(
    "https://dev-ajrt-kp3.us.auth0.com/userinfo",
    {
      headers: {
        Authorization: token,
      },
    }
  );
  return auth0Request.json();
}
```

`server/index.js` では `server/auth0.js` の関数を読み込み、 `/restaurants/:restaurantId/reviews` に対する POST リクエストのエンドポイントを実装します。まず、`app.use(express.json());` を加えることで、`Content-Type: application/json` をヘッダーに持って送られてきたメッセージのボディを JSON として解釈できるようにしておきます。

```diff:server/index.js
 import express from "express";
 import cors from "cors";
 import sequelize from "sequelize";
 import { Restaurant, Review, User } from "./models.js";
+import { checkJwt, getUser } from "./auth0.js";

 const app = express();
 app.use(cors());
+app.use(express.json());

 app.get("/restaurants", async (req, res) => {
   const limit = +req.query.limit || 5;
   const offset = +req.query.offset || 0;
   const restaurants = await Restaurant.findAndCountAll({
     attributes: {
       include: [
         [
           sequelize.literal(
             `(SELECT COUNT(*) FROM reviews AS r WHERE r.restaurant_id = restaurant.id)`
           ),
           "review_count",
         ],
       ],
     },
     include: { model: Review, limit: 3, include: { model: User } },
     order: [[sequelize.literal("review_count"), "DESC"]],
     limit,
     offset,
   });
   res.json(restaurants);
 });

 app.get("/restaurants/:restaurantId", async (req, res) => {
   const restaurantId = +req.params.restaurantId;
   const restaurant = await Restaurant.findByPk(restaurantId);
   if (!restaurant) {
     res.status(404).send("not found");
     return;
   }
   res.json(restaurant);
 });

 app.get("/restaurants/:restaurantId/reviews", async (req, res) => {
   const restaurantId = +req.params.restaurantId;
   const limit = +req.query.limit || 5;
   const offset = +req.query.offset || 0;
   const restaurant = await Restaurant.findByPk(restaurantId);
   if (!restaurant) {
     res.status(404).send("not found");
     return;
   }
   const reviews = await Review.findAndCountAll({
     include: { model: User },
     where: { restaurantId },
     limit,
     offset,
   });
   res.json(reviews);
 });

+app.post("/restaurants/:restaurantId/reviews", checkJwt, async (req, res) => {
+  const auth0User = await getUser(req.get("Authorization"));
+  const [user, created] = await User.findOrCreate({
+    where: { sub: auth0User.sub },
+    defaults: {
+      nickname: auth0User.nickname,
+    },
+  });
+  if (!created) {
+    user.nickname = auth0User.nickname;
+    await user.save();
+  }
+
+  const restaurantId = +req.params.restaurantId;
+  const restaurant = await Restaurant.findByPk(restaurantId);
+  if (!restaurant) {
+    res.status(404).send("not found");
+    return;
+  }
+
+  const record = {
+    title: req.body.title,
+    comment: req.body.comment,
+    userId: user.id,
+    restaurantId,
+  };
+
+  if (!record.title || !record.comment) {
+    res.status(400).send("bad request");
+    return;
+  }
+
+  const review = await Review.create(record);
+  res.json(review);
+});

 const port = process.env.PORT || 5000;
 app.listen(port, () => {
   console.log(`Listening at http://localhost:${port}`);
 });
```

`app.post`でエンドポイントの実装を行います。実際のリクエストを処理する `handler` の前に、第 2 引数として `checkJwt` を加えます。`checkJwt` はリクエストを読み込み正しい認証情報が付与されているかをチェックして問題がなければここで実装している `handler` に処理が渡ってきます。`handler` の中身では、Auth0 から取得したユーザー情報で `users` テーブルを更新し、リクエストが正しいか検証した後にデータベースに投稿されたレビューを記録します。データベースにレビューを記録する INSERT 文は`Review.create` によって発行されます。

## クライアントの実装

次にクライアントでの認証機能の実装です。

React で Auth0 の認証を利用するために `@auth0/auth0-react` をインストールします。`client`ディレクトリで以下のコマンドを実行しましょう。

```shell-session
$ npm i @auth0/auth0-react
```

`.env.development` と `.env.production` に Auth0 のテナントやアプリケーション、API の情報を加えます。資料の内容は例なので、自分で作成した情報を記入してください。

```diff:client/.env.development
 VITE_API_ORIGIN=http://localhost:5000
+VITE_AUTH0_DOMAIN=dev-ajrt-kp3.us.auth0.com
+VITE_AUTH0_CLIENT_ID=th264hq23cFigTYKb1r1ubAAPNvNJ4Fm
+VITE_AUTH0_AUDIENCE=https://desolate-lowlands-46852.herokuapp.com
```

```diff:client/.env.production
 VITE_API_ORIGIN=https://desolate-lowlands-46852.herokuapp.com
+VITE_AUTH0_DOMAIN=dev-ajrt-kp3.us.auth0.com
+VITE_AUTH0_CLIENT_ID=cGIzqEomOg4TLqiH6VfLWolA6gwSVzWN
+VITE_AUTH0_AUDIENCE=https://desolate-lowlands-46852.herokuapp.com
```

以下のように `client/src/main.jsx` を編集して認証機能を加えます。

```diff:client/src/main.jsx
 import "bulma/css/bulma.css";
 import { createRoot } from "react-dom/client";
+import { Auth0Provider } from "@auth0/auth0-react";
 import { App } from "./App.jsx";

-createRoot(document.querySelector("#content")).render(<App />);
+createRoot(document.querySelector("#content")).render(
+  <Auth0Provider
+    domain={import.meta.env.VITE_AUTH0_DOMAIN}
+    clientId={import.meta.env.VITE_AUTH0_CLIENT_ID}
+    redirectUri={window.location.origin}
+  >
+    <App />
+  </Auth0Provider>,
+);
```

`client/src/App.jsx` を編集して、ログインボタンが動作するようにします。すでにログイン中の場合はログアウトができるようにログアウトボタンに切り替えます。

```diff:client/src/App.jsx
 import { BrowserRouter, Route, Routes } from "react-router-dom";
+import { useAuth0 } from "@auth0/auth0-react";
 import { RootPage } from "./pages/Root.jsx";
 import { RestaurantDetailPage } from "./pages/RestaurantDetail.jsx";
 import { RestaurantListPage } from "./pages/RestaurantList.jsx";

+function AuthButton() {
+  const { isLoading, isAuthenticated, loginWithRedirect, logout } = useAuth0();
+
+  function handleClickLoginButton() {
+    loginWithRedirect({
+      appState: {
+        path: window.location.pathname,
+      },
+    });
+  }
+
+  function handleClickLogoutButton() {
+    logout({
+      localOnly: true,
+    });
+  }
+
+  if (isLoading) {
+    return (
+      <button className="button is-warning is-inverted is-outlined is-loading">
+        Loading
+      </button>
+    );
+  }
+  if (isAuthenticated) {
+    return (
+      <button
+        className="button is-warning is-inverted is-outlined"
+        onClick={handleClickLogoutButton}
+      >
+        ログアウト
+      </button>
+    );
+  }
+  return (
+    <button
+      className="button is-warning is-inverted is-outlined"
+      onClick={handleClickLoginButton}
+    >
+      ログイン
+    </button>
+  );
+}

 function Header() {
   return (
     <section className="hero is-warning">
       <div className="hero-body">
         <div className="container">
           <h1 className="title">
             日大文理
             <br className="is-hidden-tablet" />
             ラーメンレビュー
           </h1>
         </div>
       </div>
     </section>
   );
 }

 function Footer() {
   return (
     <footer className="footer ">
       <div className="content">
         <p className="has-text-centered">
           これは日本大学文理学部情報科学科の開講科目「Web
           プログラミング」の教材として作成されたサンプルアプリケーションです。
         </p>
       </div>
     </footer>
   );
 }

 export function App() {
   return (
     <BrowserRouter>
       <Header />
       <section className="section has-background-warning-light">
         <div className="container">
           <div className="block has-text-right">
-            <button className="button is-warning is-inverted is-outlined">
-              ログイン
-            </button>
+            <AuthButton />
           </div>
           <Routes>
             <Route path="/" element={<RootPage />} />
             <Route path="/restaurants" element={<RestaurantListPage />} />
             <Route
               path="/restaurants/:restaurantId"
               element={<RestaurantDetailPage />}
             />
           </Routes>
         </div>
       </section>
       <Footer />
     </BrowserRouter>
   );
 }
```

React のコンポーネントの中で `useAuth0` を呼び出すことで、現在の認証情報やログイン・ログアウトを行う関数を取り出すことができます。

次に `client/src/pages/RestaurantDetails.jsx` を以下のように編集して、レビューの投稿フォームが機能するようにします。

```diff:client/src/pages/RestaurantDetail.jsx
 import { useEffect, useState } from "react";
 import { useLocation, useParams } from "react-router-dom";
+import { useAuth0 } from "@auth0/auth0-react";
-import { getRestaurant, getRestaurantReviews } from "../api.js";
+import {
+  getRestaurant,
+  getRestaurantReviews,
+  postRestaurantReview,
+} from "../api.js";
 import { getRestaurant, getRestaurantReviews } from "../api.js";
 import { Breadcrumb, Loading, Pagination, Review } from "../components";

 function Form({ onSubmit }) {
+  const { isAuthenticated } = useAuth0();
+
   async function handleFormSubmit(event) {
     event.preventDefault();
     if (onSubmit) {
       const record = {
         title: event.target.elements.title.value,
         comment: event.target.elements.comment.value,
       };
       event.target.elements.title.value = "";
       event.target.elements.comment.value = "";
       onSubmit(record);
     }
   }

   return (
     <form onSubmit={handleFormSubmit}>
       <div className="field">
         <div className="control">
           <label className="label">タイトル</label>
           <div className="control">
-            <input name="title" className="input" required disabled />
+            <input
+              name="title"
+              className="input"
+              required
+              disabled={!isAuthenticated}
+            />
           </div>
         </div>
       </div>
       <div className="field">
         <div className="control">
           <label className="label">コメント</label>
           <div className="control">
-            <textarea name="comment" className="textarea" required disabled />
+            <textarea
+              name="comment"
+              className="textarea"
+              required
+              disabled={!isAuthenticated}
+            />
           </div>
         </div>
       </div>
       <div className="field">
         <div className="control">
-          <button type="submit" className="button is-warning" disabled>
+          <button
+            type="submit"
+            className="button is-warning"
+            disabled={!isAuthenticated}
+          >
             レビューを投稿
           </button>
         </div>
         <p className="help">ログインが必要です。</p>
       </div>
     </form>
   );
 }

 function Restaurant({ restaurant, reviews, page, perPage }) {
   return (
     <>
       <article className="box">
         <h3 className="title is-5">{restaurant.name}</h3>
         <div className="columns">
           <div className="column is-6">
             <figure className="image is-square">
               <img
                 src={restaurant.image || "/images/restaurants/noimage.png"}
                 alt={restaurant.name}
               />
             </figure>
           </div>
           <div className="column is-6">
             <figure className="image is-square">
               <div
                 className="has-ratio"
                 dangerouslySetInnerHTML={{ __html: restaurant.map }}
               ></div>
             </figure>
           </div>
         </div>
       </article>
       <div className="box">
         {reviews.rows.length === 0 ? (
           <p>レビューがまだありません。</p>
         ) : (
           <>
             <div className="block">
               <p>{reviews.count}件のレビュー</p>
             </div>
             <div className="block">
               {reviews.rows.map((review) => {
                 return <Review key={review.id} review={review} />;
               })}
             </div>
             <div className="block">
               <Pagination
                 path={`/restaurants/${restaurant.id}`}
                 page={page}
                 perPage={perPage}
                 count={reviews.count}
               />
             </div>
           </>
         )}
       </div>
     </>
   );
 }

 export function RestaurantDetailPage() {
   const [restaurant, setRestaurant] = useState(null);
   const [reviews, setReviews] = useState(null);

+  const { getAccessTokenWithPopup } = useAuth0();
+
   const params = useParams();
   const location = useLocation();
   const query = new URLSearchParams(location.search);
   const perPage = 5;
   const page = +query.get("page") || 1;

   useEffect(() => {
     getRestaurant(params.restaurantId).then((data) => {
       setRestaurant(data);
     });
   }, [params.restaurantId]);

   useEffect(() => {
     getRestaurantReviews(params.restaurantId, {
       limit: perPage,
       offset: (page - 1) * perPage,
     }).then((data) => {
       setReviews(data);
     });
   }, [params.restaurantId, page]);

+  async function handleFormSubmit(record) {
+    await postRestaurantReview(
+      params.restaurantId,
+      record,
+      getAccessTokenWithPopup
+    );
+    const data = await getRestaurantReviews(params.restaurantId, {
+      limit: perPage,
+      offset: (page - 1) * perPage,
+    });
+    setReviews(data);
+  }
+
   return (
     <>
       <div className="box">
         <Breadcrumb
           links={[
             { href: "/", content: "Top" },
             { href: "/restaurants", content: "ラーメン店一覧" },
             {
               href: `/restaurants/${params.restaurantId}`,
               content: restaurant && `${restaurant.name} の情報`,
               active: true,
             },
           ]}
         />
       </div>
       {restaurant == null || reviews == null ? (
         <Loading />
       ) : (
         <Restaurant
           restaurant={restaurant}
           reviews={reviews}
           page={page}
           perPage={perPage}
         />
       )}
       <div className="box">
-        <Form />
+        <Form onSubmit={handleFormSubmit} />
       </div>
     </>
   );
 }
```

実際に API サーバーに対して認証情報を付けて POST リクエストを行う処理を `client/src/api.js` に加えます。API にアクセスするためのトークンは `useAuth0()` から取り出した `getAccessTokenSlicently` または `getAccessTokenWithPopup` によって取得します。取得したトークンはリクエストの Authorization ヘッダーに加えてサーバーに送信します。

```diff:client/src/api.js
 async function request(path, options = {}) {
   const url = `${import.meta.env.VITE_API_ORIGIN}${path}`;
   const response = await fetch(url, options);
   return response.json();
 }

 export async function getRestaurants(arg = {}) {
   const params = new URLSearchParams(arg);
   return request(`/restaurants?${params.toString()}`);
 }

 export async function getRestaurant(restaurantId) {
   return request(`/restaurants/${restaurantId}`);
 }

 export async function getRestaurantReviews(restaurantId, arg = {}) {
   const params = new URLSearchParams(arg);
   return request(`/restaurants/${restaurantId}/reviews?${params.toString()}`);
 }

+export async function postRestaurantReview(
+  restaurantId,
+  record,
+  getAccessToken
+) {
+  const token = await getAccessToken({
+    audience: import.meta.env.VITE_AUTH0_AUDIENCE,
+  });
+  return request(`/restaurants/${restaurantId}/reviews`, {
+    body: JSON.stringify(record),
+    headers: {
+      Authorization: `Bearer ${token}`,
+      "Content-Type": "application/json",
+    },
+    method: "POST",
+  });
+}
```

ここまでで認証機能を実装することができました。http://localhost:5173 にアクセスしてログインとレビュー投稿機能が動作するか確認してみましょう。

最後にデータベースをクリアして本番環境の更新を行いましょう。データベースの初期化プログラムは、仮データからユーザーとレビューを追加していました。以下のように `server/migration.js` を修正して、ユーザーとレビューの仮データ追加処理を取り除きましょう。

```diff:server/migration.js
 import { sequelize, Restaurant, Review, User } from "./models.js";
 import * as data from "./sample-data.js";

 await sequelize.sync({ force: true });
 for (const { name, image, map } of data.restaurants) {
   await Restaurant.create({ name, image, map });
 }
-for (const { sub, nickname } of data.users) {
-  await User.create({ sub, nickname });
-}
-for (const { title, comment, userId, restaurantId } of data.reviews) {
-  await Review.create({ title, comment, userId, restaurantId });
-}
```

まずはデータベースの更新を行います。`server` ディレクトリ内で以下のコマンドを実行しましょう。

```shell-session
$ node migration.js
$ heroku pg:reset
$ heroku pg:push postgres://postgres:postgres@localhost:5432/review_app DATABASE_URL
```

続けて`server` ディレクトリ内で以下のコマンドを実行して API サーバーの更新を行います。

```shell-session
$ git add .
$ git commit -m 'update'
$ git push heroku master
```

最後に `client` ディレクトリで以下のコマンドを実行してフロントエンドの更新を行います。

```shell-session
$ npm run deploy
```

# おわりに

本チュートリアルでは、Node.js と PostgreSQL を利用した REST API と、それと連携する React アプリケーションを開発しました。アプリケーション全体を開発するためにサーバーサイド Web フレームワークの Express や OR マッパーの Sequelize、React で複数ページを扱うための React Router など様々なライブラリが登場しました。それらの機能は多く、本チュートリアルで扱ったのは極一部なので、各ライブラリのチュートリアルやドキュメントにも目を通しておくと自力で Web アプリを開発するときに役立つでしょう。また、API サーバーの公開に Heroku、ユーザー認証に Auth0 を利用しました。これらは無料でも十分に使えますが、有料プランであれば商用の Web サービスでも十分な機能を備えています。
こちらも詳細な使い方は各自で調べてみると良いでしょう。

データベースが扱えるようになると実現できる Web アプリの幅は格段に広がります。このチュートリアルアプリを改造するだけでも様々なアプリが実現できるでしょう。興味を持った人は例えば以下のような改造にチャレンジしてみてください。

- ラーメン店の説明等の追加情報を充実する
- レビューする際に得点を付けられるようにする
- ユーザーが投稿したレビューを一覧できるユーザーページを作る
- レビューやユーザーに対して「いいね」を送れるようにする
- ラーメン以外のデータを自分で用意してオリジナルのレビューサイトを作る
