### 例外フィルター

Nestには組み込みの**例外レイヤー**があり、アプリケーション全体で処理されていない例外を処理する責任を持ちます。アプリケーションコードで処理されない例外は、このレイヤーによってキャッチされ、適切なユーザーフレンドリーなレスポンスが自動的に送信されます。

<figure>
  <img class="illustrative-image" src="/assets/Filter_1.png" />
</figure>

標準では、この処理は組み込みの**グローバル例外フィルター**によって実行され、`HttpException`型(およびその派生クラス)の例外を処理します。例外が**認識されない**場合(つまり`HttpException`でもなく、`HttpException`を継承したクラスでもない場合)、組み込みの例外フィルターは以下のようなデフォルトのJSONレスポンスを生成します：

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> info **ヒント** グローバル例外フィルターは`http-errors`ライブラリを部分的にサポートしています。基本的に、`statusCode`と`message`プロパティを含むスローされた例外は、適切に処理されレスポンスとして送信されます（認識されない例外に対するデフォルトの`InternalServerErrorException`の代わりに）。

#### 標準例外のスロー

Nestは`@nestjs/common`パッケージから`HttpException`クラスを提供しています。一般的なHTTP REST/GraphQL APIベースのアプリケーションでは、特定のエラー状況が発生した時に標準的なHTTPレスポンスオブジェクトを送信することがベストプラクティスです。

例えば、`CatsController`には`findAll()`メソッド（`GET`ルートハンドラ）があります。このルートハンドラが何らかの理由で例外をスローすると仮定しましょう。これを示すために、次のようにハードコードしてみます：

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> info **ヒント** ここでは`HttpStatus`を使用しています。これは`@nestjs/common`パッケージからインポートされるヘルパー列挙型です。

クライアントがこのエンドポイントを呼び出すと、レスポンスは次のようになります：

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException`コンストラクタは、レスポンスを決定する2つの必須引数を取ります：

- `response`引数は、JSONレスポンスボディを定義します。以下で説明するように、`string`または`object`を指定できます。
- `status`引数は、[HTTPステータスコード](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)を定義します。

デフォルトでは、JSONレスポンスボディには2つのプロパティが含まれます：

- `statusCode`: デフォルトでは`status`引数で提供されたHTTPステータスコードになります
- `message`: `status`に基づいたHTTPエラーの短い説明

JSONレスポンスボディのメッセージ部分だけをオーバーライドするには、`response`引数に文字列を指定します。JSONレスポンスボディ全体をオーバーライドするには、`response`引数にオブジェクトを渡します。Nestはそのオブジェクトをシリアライズし、JSONレスポンスボディとして返します。

2番目のコンストラクタ引数 - `status` - は有効なHTTPステータスコードである必要があります。
ベストプラクティスは、`@nestjs/common`からインポートされる`HttpStatus`列挙型を使用することです。

**3番目**のコンストラクタ引数（オプション）- `options` - はエラーの[cause](https://nodejs.org/en/blog/release/v16.9.0/#error-cause)を提供するために使用できます。この`cause`オブジェクトはレスポンスオブジェクトにシリアライズされませんが、`HttpException`がスローされた原因となった内部エラーに関する貴重な情報を提供できるため、ログ記録の目的で役立ちます。

以下は、レスポンスボディ全体をオーバーライドし、エラーの原因を提供する例です：

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

上記を使用すると、レスポンスは次のようになります：

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

#### 例外のログ記録

デフォルトでは、例外フィルターは`HttpException`（およびその継承クラス）のような組み込みの例外をログに記録しません。これらの例外がスローされた場合、通常のアプリケーションフローの一部として扱われるため、コンソールには表示されません。同じ動作が`WsException`や`RpcException`などの他の組み込み例外にも適用されます。

これらの例外はすべて、`@nestjs/common`パッケージからエクスポートされる基本の`IntrinsicException`クラスを継承しています。このクラスは、通常のアプリケーション操作の一部である例外とそうでない例外を区別するのに役立ちます。

これらの例外をログに記録したい場合は、カスタム例外フィルターを作成することができます。次のセクションでその方法を説明します。

#### カスタム例外

多くの場合、カスタム例外を作成する必要はなく、次のセクションで説明する組み込みのNest HTTP例外を使用できます。カスタム例外を作成する必要がある場合は、基本の`HttpException`クラスを継承する**例外階層**を作成することがグッドプラクティスです。このアプローチでは、Nestは例外を認識し、自動的にエラーレスポンスを処理します。以下のようなカスタム例外を実装してみましょう：

```typescript
@@filename(forbidden.exception)
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

`ForbiddenException`は基本の`HttpException`を継承しているため、組み込みの例外ハンドラーとシームレスに連携します。したがって、`findAll()`メソッド内で使用することができます。

```typescript
@@filename(cats.controller)
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

#### 組み込みのHTTP例外

Nestは、基本の`HttpException`を継承した標準例外のセットを提供します。これらは`@nestjs/common`パッケージから公開され、最も一般的なHTTP例外の多くを表現します：

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

すべての組み込み例外は、`options`パラメータを使用してエラーの`cause`とエラーの説明の両方を提供することもできます：

```typescript
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
```

上記を使用すると、レスポンスは次のようになります：

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

#### 例外フィルター

基本（組み込み）の例外フィルターは多くのケースを自動的に処理できますが、例外レイヤーを**完全にコントロール**したい場合もあるでしょう。例えば、ログを追加したり、動的な要因に基づいて異なるJSONスキーマを使用したりする場合です。**例外フィルター**はまさにこの目的のために設計されています。これらにより、制御の正確なフローとクライアントに送り返されるレスポンスの内容をコントロールすることができます。

`HttpException`クラスのインスタンスである例外をキャッチし、カスタムレスポンスロジックを実装する例外フィルターを作成してみましょう。そのためには、基盤となるプラットフォームの`Request`オブジェクトと`Response`オブジェクトにアクセスする必要があります。`Request`オブジェクトにアクセスして元の`url`を取得し、ログ情報に含めます。`Response`オブジェクトを使用して、`response.json()`メソッドを使用して送信されるレスポンスを直接制御します。

```typescript
@@filename(http-exception.filter)
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
@@switch
import { Catch, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter {
  catch(exception, host) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> info **ヒント** すべての例外フィルターは、ジェネリックな`ExceptionFilter<T>`インターフェースを実装する必要があります。これにより、指定された署名を持つ`catch(exception: T, host: ArgumentsHost)`メソッドを提供する必要があります。`T`は例外の型を示します。

> warning **警告** `@nestjs/platform-fastify`を使用している場合は、`response.json()`の代わりに`response.send()`を使用できます。`fastify`から正しい型をインポートすることを忘れないでください。

`@Catch(HttpException)`デコレーターは、必要なメタデータを例外フィルターにバインドし、このフィルターが`HttpException`型の例外のみを探していることをNestに伝えます。`@Catch()`デコレーターは単一のパラメータ、またはコンマで区切られたリストを取ることができます。これにより、一度に複数の種類の例外に対してフィルターを設定することができます。

#### 引数ホスト

`catch()`メソッドのパラメータを見てみましょう。`exception`パラメータは、現在処理中の例外オブジェクトです。`host`パラメータは`ArgumentsHost`オブジェクトです。`ArgumentsHost`は強力なユーティリティオブジェクトで、[実行コンテキストの章](/fundamentals/execution-context)でさらに詳しく説明します\*。このコードサンプルでは、例外が発生した元のリクエストハンドラ（コントローラ内）に渡される`Request`オブジェクトと`Response`オブジェクトへの参照を取得するために使用しています。このコードサンプルでは、`ArgumentsHost`のヘルパーメソッドを使用して、必要な`Request`オブジェクトと`Response`オブジェクトを取得しています。`ArgumentsHost`についての詳細は[こちら](/fundamentals/execution-context)をご覧ください。

\*この抽象化レベルが必要な理由は、`ArgumentsHost`がすべてのコンテキスト（現在作業しているHTTPサーバーコンテキストだけでなく、マイクロサービスやWebSocketsも）で機能するためです。実行コンテキストの章では、`ArgumentsHost`とそのヘルパー関数の力を使用して、**任意の**実行コンテキストに対して適切な<a href="https://docs.nestjs.com/fundamentals/execution-context#host-methods">基礎となる引数</a>にアクセスする方法を見ていきます。これにより、すべてのコンテキストで動作する汎用的な例外フィルターを作成することができます。

<app-banner-courses></app-banner-courses>

#### フィルターのバインド

新しい`HttpExceptionFilter`を`CatsController`の`create()`メソッドにバインドしてみましょう。

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(new HttpExceptionFilter())
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **ヒント** `@UseFilters()`デコレータは`@nestjs/common`パッケージからインポートされています。

ここでは`@UseFilters()`デコレータを使用しています。`@Catch()`デコレータと同様に、単一のフィルターインスタンス、またはコンマで区切られたフィルターインスタンスのリストを取ることができます。ここでは、`HttpExceptionFilter`のインスタンスをその場で作成しています。また、インスタンスの代わりにクラスを渡すこともできます。これにより、インスタンス化の責任をフレームワークに委ね、**依存性の注入**を可能にします。

```typescript
@@filename(cats.controller)
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
@@switch
@Post()
@UseFilters(HttpExceptionFilter)
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> info **ヒント** 可能な限り、インスタンスの代わりにクラスを使用してフィルターを適用することを推奨します。Nestは同じクラスのインスタンスをモジュール全体で簡単に再利用できるため、**メモリ使用量**が削減されます。

上記の例では、`HttpExceptionFilter`は単一の`create()`ルートハンドラにのみ適用され、メソッドスコープとなっています。例外フィルターは異なるレベルでスコープを設定できます：コントローラ/リゾルバ/ゲートウェイのメソッドスコープ、コントローラスコープ、またはグローバルスコープです。
例えば、フィルターをコントローラスコープとして設定するには、次のようにします：

```typescript
@@filename(cats.controller)
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

この構成により、`CatsController`内で定義されたすべてのルートハンドラに対して`HttpExceptionFilter`が設定されます。

グローバルスコープのフィルターを作成するには、次のようにします：

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **警告** `useGlobalFilters()`メソッドは、ゲートウェイやハイブリッドアプリケーションに対してフィルターを設定しません。

グローバルスコープのフィルターは、すべてのコントローラとすべてのルートハンドラに対して、アプリケーション全体で使用されます。依存性注入に関しては、任意のモジュールの外部で登録されたグローバルフィルター（上記の例のように`useGlobalFilters()`を使用）は、これがモジュールのコンテキスト外で行われるため、依存性を注入することができません。この問題を解決するために、以下の構成を使用して、**任意のモジュールから直接**グローバルスコープのフィルターを登録することができます：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> info **ヒント** このアプローチを使用してフィルターに対して依存性注入を実行する場合、この構成が使用されるモジュールに関係なく、フィルターは実際にグローバルであることに注意してください。どこで行うべきでしょうか？フィルター（上記の例では`HttpExceptionFilter`）が定義されているモジュールを選択してください。また、`useClass`はカスタムプロバイダー登録の唯一の方法ではありません。詳細は[こちら](/fundamentals/custom-providers)をご覧ください。

この技術を使用して、必要な数のフィルターを追加できます。単にprovidersの配列に各フィルターを追加するだけです。

#### すべてをキャッチ

処理されていない**すべての**例外をキャッチするには（例外の型に関係なく）、`@Catch()`デコレータのパラメータリストを空のままにします（例：`@Catch()`）。

以下の例では、プラットフォームに依存しないコードを示しています。これは[HTTPアダプター](./faq/http-adapter)を使用してレスポンスを配信し、プラットフォーム固有のオブジェクト（`Request`および`Response`）を直接使用しないためです：

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // 特定の状況では、コンストラクタメソッドで`httpAdapter`が利用できない
    // 場合があるため、ここで解決する必要があります。
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> warning **警告** すべてをキャッチする例外フィルターを特定の型にバインドされたフィルターと組み合わせる場合、「すべてをキャッチ」フィルターを最初に宣言して、特定のフィルターがバインドされた型を正しく処理できるようにする必要があります。

#### 継承

通常、アプリケーションの要件を満たすように完全にカスタマイズされた例外フィルターを作成します。しかし、組み込みのデフォルトの**グローバル例外フィルター**を単純に拡張し、特定の要因に基づいて動作をオーバーライドしたい場合もあるでしょう。

例外処理を基本フィルターに委譲するには、`BaseExceptionFilter`を拡張し、継承された`catch()`メソッドを呼び出す必要があります。

```typescript
@@filename(all-exceptions.filter)
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

> warning **警告** `BaseExceptionFilter`を拡張するメソッドスコープおよびコントローラスコープのフィルターは、`new`でインスタンス化すべきではありません。代わりに、フレームワークに自動的にインスタンス化させてください。

グローバルフィルターは基本フィルターを拡張**できます**。これは2つの方法で行うことができます。

最初の方法は、カスタムグローバルフィルターをインスタンス化する際に`HttpAdapter`参照を注入することです：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

2番目の方法は、`APP_FILTER`トークンを使用することです（<a href="exception-filters#binding-filters">ここで示したように</a>）。
