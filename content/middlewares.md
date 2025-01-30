### ミドルウェア

ミドルウェアは、ルートハンドラーの**前**に呼び出される関数です。ミドルウェア関数は、アプリケーションのリクエスト-レスポンスサイクルにおいて[request](https://expressjs.com/en/4x/api.html#req)オブジェクトと[response](https://expressjs.com/en/4x/api.html#res)オブジェクト、そして`next()`ミドルウェア関数にアクセスできます。**next**ミドルウェア関数は、一般的に`next`という名前の変数で表されます。

<figure><img class="illustrative-image" src="/assets/Middlewares_1.png" /></figure>

Nestのミドルウェアは、デフォルトで[express](https://expressjs.com/en/guide/using-middleware.html)のミドルウェアと同等です。Express公式ドキュメントからの以下の説明は、ミドルウェアの機能を示しています：

<blockquote class="external">
  ミドルウェア関数は以下のタスクを実行できます：
  <ul>
    <li>任意のコードを実行する。</li>
    <li>requestオブジェクトとresponseオブジェクトに変更を加える。</li>
    <li>リクエスト-レスポンスサイクルを終了する。</li>
    <li>スタック内の次のミドルウェア関数を呼び出す。</li>
    <li>現在のミドルウェア関数がリクエスト-レスポンスサイクルを終了しない場合、次のミドルウェア関数に制御を渡すために<code>next()</code>を呼び出す必要があります。そうしないと、リクエストは宙ぶらりんの状態になります。</li>
  </ul>
</blockquote>

カスタムNestミドルウェアは、関数として、または`@Injectable()`デコレータを持つクラスとして実装できます。クラスは`NestMiddleware`インターフェースを実装する必要がありますが、関数には特別な要件はありません。クラスメソッドを使用して、シンプルなミドルウェア機能を実装してみましょう。

> warning **警告** `Express`と`fastify`はミドルウェアの扱い方が異なり、異なるメソッドシグネチャを提供します。詳細は[こちら](/techniques/performance#middleware)をご覧ください。

```typescript
@@filename(logger.middleware)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('Request...');
    next();
  }
}
```

#### 依存性の注入

Nestミドルウェアは依存性の注入を完全にサポートしています。プロバイダーやコントローラーと同様に、同じモジュール内で利用可能な依存関係を**注入**することができます。通常通り、これは`constructor`を通じて行われます。

#### ミドルウェアの適用

`@Module()`デコレータにはミドルウェアの場所はありません。代わりに、モジュールクラスの`configure()`メソッドを使用してセットアップします。ミドルウェアを含むモジュールは`NestModule`インターフェースを実装する必要があります。`AppModule`レベルで`LoggerMiddleware`をセットアップしてみましょう。

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

上記の例では、`CatsController`内で以前定義した`/cats`ルートハンドラーに対して`LoggerMiddleware`をセットアップしています。また、ミドルウェアを設定する際に`forRoutes()`メソッドにルートの`path`とリクエストの`method`を含むオブジェクトを渡すことで、特定のリクエストメソッドにミドルウェアを制限することもできます。以下の例では、目的のリクエストメソッドタイプを参照するために`RequestMethod`列挙型をインポートしていることに注目してください。

```typescript
@@filename(app.module)
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
@@switch
import { Module, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> info **ヒント** `configure()`メソッドは`async/await`を使用して非同期にすることができます（例：`configure()`メソッド本体内で非同期操作の完了を`await`できます）。

> warning **警告** `express`アダプターを使用する場合、NestJSアプリはデフォルトで`body-parser`パッケージから`json`と`urlencoded`を登録します。これは、`MiddlewareConsumer`を通じてそのミドルウェアをカスタマイズしたい場合、`NestFactory.create()`でアプリケーションを作成する際に`bodyParser`フラグを`false`に設定してグローバルミドルウェアをオフにする必要があることを意味します。

#### ルートのワイルドカード

NestJSミドルウェアではパターンベースのルートもサポートされています。例えば、名前付きワイルドカード（`*splat`）を使用して、ルート内の任意の文字の組み合わせにマッチさせることができます。以下の例では、後続の文字数に関係なく、`abcd/`で始まる任意のルートに対してミドルウェアが実行されます。

```typescript
forRoutes({
  path: 'abcd/*splat',
  method: RequestMethod.ALL,
});
```

> info **ヒント** `splat`はワイルドカードパラメータの単なる名前で、特別な意味はありません。例えば`*wildcard`のように、好きな名前を付けることができます。

`'abcd/*'`というルートパスは、`abcd/1`、`abcd/123`、`abcd/abc`などにマッチします。ハイフン（`-`）とドット（`.`）は文字列ベースのパスでは文字通りに解釈されます。ただし、追加の文字がない`abcd/`はルートにマッチしません。これには、ワイルドカードを中括弧で囲んでオプショナルにする必要があります：

```typescript
forRoutes({
  path: 'abcd/{*splat}',
  method: RequestMethod.ALL,
});
```

#### ミドルウェアコンシューマー

`MiddlewareConsumer`はヘルパークラスです。ミドルウェアを管理するための複数のビルトインメソッドを提供します。これらはすべて[フルエントスタイル](https://en.wikipedia.org/wiki/Fluent_interface)で**チェーン**することができます。`forRoutes()`メソッドは、単一の文字列、複数の文字列、`RouteInfo`オブジェクト、コントローラークラス、さらには複数のコントローラークラスを受け取ることができます。ほとんどの場合、おそらくカンマで区切られた**コントローラー**のリストを渡すことになるでしょう。以下は単一のコントローラーを使用した例です：

```typescript
@@filename(app.module)
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
@@switch
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> info **ヒント** `apply()`メソッドは単一のミドルウェアを取ることも、<a href="/middleware#multiple-middleware">複数のミドルウェア</a>を指定するための複数の引数を取ることもできます。

#### ルートの除外

場合によっては、特定のルートにミドルウェアを適用したくない場合があります。これは`exclude()`メソッドを使用して簡単に実現できます。`exclude()`メソッドは、除外するルートを識別するために、単一の文字列、複数の文字列、または`RouteInfo`オブジェクトを受け取ります。

使用例は以下の通りです：

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/{*splat}',
  )
  .forRoutes(CatsController);
```

> info **ヒント** `exclude()`メソッドは[path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters)パッケージを使用してワイルドカードパラメータをサポートしています。

上記の例では、`LoggerMiddleware`は`CatsController`内で定義されたすべてのルートに適用されますが、`exclude()`メソッドに渡された3つのルートは**除外**されます。

このアプローチにより、特定のルートやルートパターンに基づいてミドルウェアを適用または除外する柔軟性が提供されます。

#### 関数型ミドルウェア

これまで使用してきた`LoggerMiddleware`クラスはとてもシンプルです。メンバー、追加のメソッド、依存関係がありません。なぜクラスの代わりにシンプルな関数として定義できないのでしょうか？実際、そうすることができます。このタイプのミドルウェアは**関数型ミドルウェア**と呼ばれます。違いを説明するために、ロガーミドルウェアをクラスベースから関数型ミドルウェアに変換してみましょう：

```typescript
@@filename(logger.middleware)
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
@@switch
export function logger(req, res, next) {
  console.log(`Request...`);
  next();
};
```

そして`AppModule`内で使用します：

```typescript
@@filename(app.module)
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> info **ヒント** ミドルウェアが依存関係を必要としない場合は、より簡単な**関数型ミドルウェア**の代替手段を検討してください。

#### 複数のミドルウェア

上述の通り、順番に実行される複数のミドルウェアをバインドするには、`apply()`メソッド内でカンマで区切ったリストを提供するだけです：

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

#### グローバルミドルウェア

登録されたすべてのルートに一度にミドルウェアをバインドしたい場合は、`INestApplication`インスタンスが提供する`use()`メソッドを使用できます：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(process.env.PORT ?? 3000);
```

> info **ヒント** グローバルミドルウェアでDIコンテナにアクセスすることはできません。`app.use()`を使用する場合は、代わりに[関数型ミドルウェア](middleware#functional-middleware)を使用できます。または、クラスミドルウェアを使用して、`AppModule`（または他のモジュール）内で`.forRoutes('*')`と共に消費することもできます。

> info **ヒント** グローバルミドルウェアでDIコンテナにアクセスすることはできません。`app.use()`を使用する場合は、代わりに[関数型ミドルウェア](middleware#functional-middleware)を使用できます。あるいは、クラスミドルウェアを使用し、`AppModule`（または他のモジュール）内で`.forRoutes('*')`と共に使用することもできます。

以上で、NestJSのミドルウェアに関する説明の全文を翻訳し終えました。このドキュメントでは、以下の主要なトピックをカバーしました：

- ミドルウェアの基本概念と機能
- クラスベースと関数型のミドルウェアの実装方法
- 依存性の注入のサポート
- ミドルウェアの適用方法とルート設定
- ワイルドカードルートの使用
- ミドルウェアコンシューマーの使用法
- ルートの除外機能
- 複数のミドルウェアの適用
- グローバルミドルウェアの設定

これらの機能を使用することで、アプリケーションのリクエスト処理パイプラインを効果的にカスタマイズすることができます。各機能は異なるユースケースに対応し、アプリケーションの要件に応じて柔軟に組み合わせることができます。

NestJSミドルウェアの主要なユースケースをまとめます：

1. リクエストのログ記録
- リクエストの詳細（メソッド、URL、ヘッダー、IPアドレスなど）の記録
- アプリケーションの監視やデバッグに活用
- パフォーマンス計測（リクエスト処理時間の記録）

2. 認証・認可
- JWTトークンの検証
- APIキーの確認
- ユーザーの権限チェック
- セッション管理

3. リクエスト前処理
- リクエストボディの変換や正規化
- カスタムヘッダーの追加
- リクエストの検証
- データの型変換

4. セキュリティ対策
- CORS設定
- CSRFトークン検証
- レート制限（Rate limiting）の実装
- XSS対策
- SQLインジェクション対策

5. キャッシュ制御
- レスポンスのキャッシュ
- 条件付きリクエストの処理
- キャッシュヘッダーの管理

6. エラーハンドリング
- グローバルなエラートラップ
- エラーのフォーマット統一
- カスタムエラーレスポンスの生成

7. データ圧縮
- レスポンスの圧縮
- 大きなデータの効率的な転送

8. 言語・地域対応
- 多言語対応（i18n）
- タイムゾーン調整
- 地域固有の設定適用

9. リクエスト制御
- リクエストのサイズ制限
- ファイルアップロードの制御
- リクエストのタイムアウト設定

10. メトリクス収集
- アプリケーションの利用統計
- パフォーマンスモニタリング
- ユーザー行動の分析

これらのユースケースは、アプリケーションの要件に応じて組み合わせることができ、クラスベースまたは関数型のミドルウェアとして実装できます。また、特定のルートに限定したり、グローバルに適用したりすることで、柔軟な制御が可能です。
