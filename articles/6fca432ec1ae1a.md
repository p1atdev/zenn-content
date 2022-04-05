---
title: "Strapi v4をHerokuとCloudinaryを使って無料で動かす"
emoji: "🌤️"
type: "tech"
topics: ["heroku", "strapi", "cloudinary"]
published: true
---

※Herokuについての知識が多少ある前提で進めます。Herokuについて知りたい場合は他の記事を参照してください。

# Strapiとは？
![](https://storage.googleapis.com/zenn-user-upload/23b4bcccfa2b-20220107.png)

https://strapi.io/

Strapiは簡単に言うと、オープンソースのセルフホスト型ヘッドレスCMSです。
CMSにしては珍しく、自分でビルドしてホストするタイプのものとなっています。
最近メジャーバージョンがv4となり、UIやビルドシステムが一新されてより使いやすくなったので紹介しようと思います。

## 機能&料金
![](https://storage.googleapis.com/zenn-user-upload/0c9cac010a7b-20220107.png)
データベースやサーバーを自分で用意する必要がある代わりに、基本的な機能は全て無料となっています。リレーションや画像の配信も無料プランでできますし、ユーザーの権限設定も細かく設定できます。GraphQLにも対応していたり(らしい？[^1])、リッチテキストをマークダウンのまま返してくれたりとContentfulやmicroCMSにない機能もあります。
特に有難いのがリッチテキストなどで**日本語が自然入力にできる**ところです。(Contentfulの日本語入力はかなりバグっていて使いづらいのです)
自分でホストするので独自ドメインが使えるという点も嬉しい。
[^1]: v4では2022/1/7現在ではGraphQLプラグインをインストールできませんが、v3までは使えたので今後対応されると思います。

# 無料で使う方法
Strapiを使うにはまずそれを動かすサーバーを用意する必要があります。選択肢としては、AWS EC2やAzure、GAE、DigitalOceanがありますが、**Heroku**を使うことで完全無料[^2]で使うことができます。
公式ドキュメントにHerokuでの[導入記事](https://docs.strapi.io/developer-docs/latest/setup-deployment-guides/deployment/hosting-guides/heroku.html)があります。Herokuは専用のPostgresSQLを使うことができるので、データベースは気にしなくて良い[^2]のですが、画像などのファイルだけはHerokuのephemeralという機能によって一定時間で綺麗に抹消されてしまうので、画像は別の場所に保存する必要があります。それに使うのが**Cloudinary**というサービスです。

[^2]:HerokuのPostgresSQLフリープランは、最大で1万行、1GBという容量制限があるので、これよりも多くのデータを保存したい場合は有料プランになる必要があります。[料金](https://elements.heroku.com/addons/heroku-postgresql)

# Cloudinary
@[card](https://cloudinary.com/)
Cloudinaryは画像の配信サーバーで、画像の変換に使うこともできますが今回は単に画像を保存するためだけに使います。

## 料金

![](https://storage.googleapis.com/zenn-user-upload/00f7f1ed8e33-20220108.png =250x)
一ヶ月で2万5千回の画像変換(今回は使わないので気にしなくていい)、25GB分のストレージ、25GB分の通信帯域がフリープランの範囲です。個人で使う分なら余裕で余ると思います。

## アカウント作成とキーの保存
アカウントを作っていない人はサクッと作ってしまいましょう。
@[card](https://cloudinary.com/users/register/free)
登録ができたら[ダッシュボード](https://cloudinary.com/console)に行き、`Cloud Name`、`API Key`、`API Secret`が後で必要になるので保存しておきます。
![](https://storage.googleapis.com/zenn-user-upload/803ec3ec8890-20220108.png)
*※2022/1/8時点でのダッシュボードの画面*

# Strapiプロジェクトの作成
基本的には[公式の説明](https://docs.strapi.io/developer-docs/latest/setup-deployment-guides/deployment/hosting-guides/heroku.html)と同じです。

## 前提
- Herokuアカウント(フリープラン以上)
- [Heroku CLI](https://devcenter.heroku.com/ja/articles/heroku-cli)
- Node.js
- Git

## Strapiプロジェクトの作成
お好きなディレクトリで以下を実行
```bash
npx create-strapi-app@latest my-project --quickstart
```
`my-project`は自分の好きな名前に変えてもいいです。`--quickstart`オプションをつけると、プロジェクト作成後にすぐにStrapiの管理画面を開くことができます。不要ならつけなくてもいいです。

## Herokuにpush
`my-project`ディレクトリが作成されるので、
```bash:./my-project
git init
git add .
git commit -m "first commit"
```
でコミットし、お好きな方法でHerokuにプッシュします。

まだHerokuのappを作成していない場合は、
```bash:./my-project
heroku login
```
を実行してHerokuにログインし、
```bash:./my-project
heroku create custom-project-name
```
で`custom-project-name`という名前のappを作成できます。この`custom-project-name`は固有の名前でないといけません。

## Heroku PostgresSQLを有効化
GUI、CUIどちらから追加しても構いません。
今回はCUIから追加します。
既に作成したappに追加する場合は、
```bash:./my-project
heroku git:remote -a your-heroku-app-name
```
を実行することで`your-heroku-app-name`を選択することができます。

次に、
```bash:./my-project
heroku addons:create heroku-postgresql:hobby-dev
```
を実行することでappにHeroku PostgresSQLが追加されます。

## 必要ライブラリのインストール
今回はnpmを使います。yarnの人は適宜置き換えてください。

HerokuのPostgresSQLを使うために必要なライブラリをインストールします。
```bash:./my-project
npm install pg pg-connection-string --save
```

## データベースを利用するためのセットアップ
以下を実行してHerokuの`NODE_ENV`を`production`にセットします。(理由は後述)
```bash:./my-project
heroku config:set NODE_ENV=production
```

Herokuで実行する際にHeroku PostgresSQLを使用するように設定をします。
`my-project`プロジェクト内に`config/env/production/database.js`というファイルを作成し、以下の内容を記述します。丸々コピペでいいです。
```js:./my-project/config/env/production/database.js
const parse = require('pg-connection-string').parse;
const config = parse(process.env.DATABASE_URL);

module.exports = ({ env }) => ({
  connection: {
    client: 'postgres',
    connection: {
      host: config.host,
      port: config.port,
      database: config.database,
      user: config.user,
      password: config.password,
      ssl: {
        rejectUnauthorized: false
      },
    },
    debug: false,
  },
});
```

次に、Heroku PostgresSQLのURLを読み込むための設定をします。
以下を実行してSQLのURLをHerokuの環境変数にセットします。丸々コピペして実行してください。
```bash:./my-project
heroku config:set MY_HEROKU_URL=$(heroku info -s | grep web_url | cut -d= -f2)
```
`config/env/production/server.js`というファイルを作成し、以下の内容を記述します。こちらもそのままコピペでいいです。
```js:./my-project/config/env/production/server.js
module.exports = ({ env }) => ({
  url: env('MY_HEROKU_URL'),
  app: {
    keys: env.array('APP_KEYS'),
  },
});
```
:::details 環境変数`NODE_ENV`に`production`をセットした理由
Strapiではセキュリティ上の理由から、公開リンクでは`production`モードにすることが推奨されています。`production`モードではContent-Type Builderで新たなContent-Typeを作成することができません。Content-Typeは記事やユーザーなどの構造体のことで、これを編集したり作成したい場合はローカル環境で編集し再度デプロイする必要があります。記事を書いたりするときにはそのまま作成できるので記事を書くたびにデプロイする必要はありません。

`config/env/production`ディレクトリ内のファイルは、`NODE_ENV`が`production`の時にのみ実行されます。
:::
:::details 2022/3/21 変更点
```js
  app: {
    keys: env.array('APP_KEYS'),
  },
```
を追記しました。
Heroku側にも `App_KEYS` をセットする必要があります。
ここはよくわからなかったのですが、 `.env` ファイルで生成されるものをそのまま移したら動きました。
:::

## 動作確認
以上が完了したら一度Herokuにデプロイして動作を確認しましょう。
`https://[your-herokuapp-name].herokuapp.com`にアクセスするとStrapiと対面できると思います。
![](https://storage.googleapis.com/zenn-user-upload/79b056e27115-20220108.png)
管理画面のURLは`https://[your-herokuapp-name].herokuapp.com/admin`です。一番最初だけアカウントの登録画面となりますが、2回目以降はログイン画面が出てきます。Wordpressみたいな感じ。
ログインすると下の画像みたいな画面になります。
![](https://storage.googleapis.com/zenn-user-upload/0013d3f7f14d-20220108.png)
今回の記事はStrapiの使い方解説ではないので詳しい使い方は解説しません。UIがわかりやすいので他のCMSを使ったことがある人ならすぐに使えるかもしません。わからなかったら[公式ドキュメント](https://docs.strapi.io/developer-docs/latest/getting-started/introduction.html)を参照してください。

# Cloudinaryと連携する
Cloudinaryと連携するために以下の作業を行います。

## ライブラリのインストール
`@strapi/provider-upload-cloudinary`をインストールします。Strapi v3についての記事では別のライブラリを使っていることがありますが、v4では`@strapi/provider-upload-cloudinary`でないとビルドに失敗します。
```bash:./my-project
npm install @strapi/provider-upload-cloudinary
```

## 環境変数のセット
最初にメモしたCloudinaryのAPIキーなどをHerokuの環境変数にセットします。
`CLOUDINARY_NAME`に`Cloud Name`を、`CLOUDINARY_KEY`に`API Key`を、`CLOUDINARY_SECRET`に`API Secret`をセットします。

## Cloudinaryライブラリの利用

`config/env/production/plugins.js`を作成し、以下のように記述します。
画像をアップロード時に自動でCloudinaryにアップロードされるようになります。
```js:./my-project/config/env/production/plugins.js
module.exports = ({ env }) => ({
  upload: {
    config: {
      provider: "cloudinary",
      providerOptions: {
        cloud_name: env("CLOUDINARY_NAME"),
        api_key: env("CLOUDINARY_KEY"),
        api_secret: env("CLOUDINARY_SECRET"),
      },
      actionOptions: {
        upload: {},
        delete: {},
      },
    },
  },
});
```

`config/env/production/middlewares.js`を作成し、以下のように記述します。
Strapi上で画像ファイルのサムネイルを表示できるようになります。
```js:./my-project/config/middlewares.js
module.exports = [
  "strapi::errors",
  {
    name: "strapi::security",
    config: {
      contentSecurityPolicy: {
        useDefaults: true,
        directives: {
          "connect-src": ["'self'", "https:"],
          "img-src": ["'self'", "data:", "blob:", "res.cloudinary.com"],
          "media-src": ["'self'", "data:", "blob:", "res.cloudinary.com"],
          upgradeInsecureRequests: null,
        },
      },
    },
  },
  "strapi::cors",
  "strapi::poweredBy",
  "strapi::logger",
  "strapi::query",
  "strapi::body",
  "strapi::session",
  "strapi::favicon",
  "strapi::public",
];
```

:::details 2022/3/21変更点
```js
    "strapi::session",
```
を追加
:::


参考
@[card](https://github.com/ebitsdev/strapi-v4-cloudinary)

## 動作確認
以上が完了したらHerokuにデプロイし、StrapiのMedia LibraryからちゃんとCloudinaryにアップロードされるかチェックしましょう。
![](https://storage.googleapis.com/zenn-user-upload/bc6fadb204eb-20220108.png)
Upload Assetsをクリックし、ワニの画像をアップロードします。

![](https://storage.googleapis.com/zenn-user-upload/4bfd1b5f7192-20220108.gif)

このGIFのようにアップロードに少し時間がかかっていたらおそらく成功です。CloudinaryのMedia Libraryを確認してみましょう。

![](https://storage.googleapis.com/zenn-user-upload/2b63b8ac26fe-20220108.png)
*sampleの画像は初めから入っているサンプル画像なので気にしないでください*

ワニの画像がちゃんと追加されています！しかし、追加したのは1枚だけなのになぜか2枚ありますね。
これは、画像のサイズによって小さいサイズのバージョンや、サムネ用サイズの画像も自動で生成してくれるためです。
画像を削除するときはStrapiからDeleteするだけでCloudinary側の画像もちゃんと削除されます。

以上で、HerokuとCloudinaryを使って無料でStrapiをホストすることができました！

# 最後に
StrapiはContentful~~なんか~~と比べてデベロッパーに優しい感じになっていて好きです。
Nuxt3はnuxt/contentが動かない(2022/1/8現在)ので、代わりにStrapiを使ってみようと思います。いい感じに行けたら記事にする予定です。

誤字や間違っている点などがあったら教えてください。