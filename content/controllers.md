### コントローラー

コントローラーは、クライアントからの**リクエスト**を処理し、**レスポンス**を返す役割を担っています。

<figure><img class="illustrative-image" src="/assets/Controllers_1.png" /></figure>

コントローラーの目的は、アプリケーションの特定のリクエストを処理することです。**ルーティング**メカニズムによって、どのコントローラーがどのリクエストを処理するかが決定されます。通常、コントローラーは複数のルートを持ち、各ルートは異なるアクションを実行できます。

基本的なコントローラーを作成するには、クラスと**デコレータ**を使用します。デコレータは、必要なメタデータをクラスに関連付け、Nest がリクエストを対応するコントローラーに接続するルーティングマップを作成できるようにします。

> info **ヒント** 組み込みの[バリデーション](https://docs.nestjs.com/techniques/validation)を備えた CRUD コントローラーを素早く作成するには、CLI の[CRUD ジェネレーター](https://docs.nestjs.com/recipes/crud-generator#crud-generator)を使用できます：`nest g resource [name]`

#### ルーティング

次の例では、基本的なコントローラーを定義するために**必須**の`@Controller()`デコレータを使用します。オプションのルートパスプレフィックスとして`cats`を指定します。`@Controller()`デコレータでパスプレフィックスを使用すると、関連するルートをグループ化し、繰り返しコードを減らすことができます。例えば、cat エンティティとの対話を管理するルートを`/cats`パスの下にグループ化したい場合、`@Controller()`デコレータで`cats`パスプレフィックスを指定できます。これにより、ファイル内の各ルートでそのパスの部分を繰り返す必要がなくなります。

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

> info **ヒント** CLI を使用してコントローラーを作成するには、単純に `$ nest g controller [name]` コマンドを実行します。

`findAll()`メソッドの前に配置された`@Get()`HTTP リクエストメソッドデコレータは、HTTP リクエストの特定のエンドポイント用のハンドラーを作成するよう Nest に指示します。このエンドポイントは、HTTP リクエストメソッド（この場合は GET）とルートパスによって定義されます。では、ルートパスとは何でしょうか？ハンドラーのルートパスは、コントローラーに宣言された（オプションの）プレフィックスと、メソッドのデコレータで指定された任意のパスを組み合わせて決定されます。すべてのルートにプレフィックス（`cats`）を設定し、メソッドデコレータに特定のパスを追加していないため、Nest は`GET /cats`リクエストをこのハンドラーにマッピングします。

前述のように、ルートパスには、オプションのコントローラーパスプレフィックス**と**メソッドのデコレータで指定された任意のパス文字列の両方が含まれます。例えば、コントローラーのプレフィックスが`cats`で、メソッドデコレータが`@Get('breed')`の場合、結果のルートは`GET /cats/breed`となります。

上記の例では、このエンドポイントに GET リクエストが行われると、Nest はリクエストをユーザー定義の`findAll()`メソッドにルーティングします。ここで選択したメソッド名は完全に任意であることに注意してください。ルートにバインドするメソッドを宣言する必要はありますが、Nest はメソッド名に特別な意味を持たせていません。

このメソッドは、関連するレスポンス（この場合は単なる文字列）とともに 200 ステータスコードを返します。なぜこれが起こるのでしょうか？説明するために、まず Nest がレスポンスを操作するために 2 つの**異なる**オプションを使用するという概念を紹介する必要があります：

はい、テーブルの部分を翻訳いたします。

<table>
  <tr>
    <td>標準的な方法（推奨）</td>
    <td>
      この組み込みメソッドを使用すると、リクエストハンドラーが JavaScript のオブジェクトや配列を返す場合、**自動的に** JSON にシリアライズされます。ただし、JavaScript のプリミティブ型（例：<code>string</code>、<code>number</code>、<code>boolean</code>）を返す場合、Nest はシリアライズを試みずに値をそのまま送信します。これによりレスポンス処理がシンプルになります：値を返すだけで、残りは Nest が処理します。
      <br />
      <br />さらに、レスポンスの**ステータスコード**はデフォルトで常に 200 となります。ただし、POST リクエストの場合は 201 が使用されます。この動作は、ハンドラーレベルで<code>@HttpCode(...)</code>デコレータを追加することで簡単に変更できます（<a href='controllers#status-code'>ステータスコード</a>を参照）。
    </td>
  </tr>
  <tr>
    <td>ライブラリ固有の方法</td>
    <td>
      ライブラリ固有の（例：Express）<a href="https://expressjs.com/en/api.html#res" rel="nofollow" target="_blank">レスポンスオブジェクト</a>を使用することができます。これはメソッドハンドラーのシグネチャで<code>@Res()</code>デコレータを使用して注入できます（例：<code>findAll(@Res() response)</code>）。このアプローチでは、そのオブジェクトが提供するネイティブのレスポンス処理メソッドを使用できます。例えば、Express では<code>response.status(200).send()</code>のようなコードでレスポンスを構築できます。
    </td>
  </tr>
</table>

> warning **警告** Nest はハンドラーが`@Res()`または`@Next()`を使用しているかを検出し、ライブラリ固有のオプションを選択したことを示します。両方のアプローチを同時に使用すると、標準的なアプローチはこの単一のルートに対して**自動的に無効**になり、期待通りに動作しなくなります。両方のアプローチを同時に使用する場合（例えば、レスポンスオブジェクトを注入してクッキー/ヘッダーの設定のみを行い、残りはフレームワークに任せる場合）、`@Res({{ '{' }} passthrough: true {{ '}' }})`デコレータで`passthrough`オプションを`true`に設定する必要があります。

#### リクエストオブジェクト

ハンドラーは、クライアントの**リクエスト**の詳細にアクセスする必要がよくあります。Nest は基盤となるプラットフォーム（デフォルトでは Express）から[リクエストオブジェクト](https://expressjs.com/en/api.html#req)へのアクセスを提供します。ハンドラーのシグネチャで`@Req()`デコレータを使用して Nest に注入を指示することで、リクエストオブジェクトにアクセスできます。

```typescript
@@filename(cats.controller)
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Bind, Get, Req } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Bind(Req())
  findAll(request) {
    return 'This action returns all cats';
  }
}
```

> info **ヒント** `express`の型定義（上記の`request: Request`パラメータの例のような）を活用するには、`@types/express`パッケージをインストールしてください。

リクエストオブジェクトは HTTP リクエストを表し、クエリ文字列、パラメータ、HTTP ヘッダー、ボディのプロパティを含んでいます（詳細は[こちら](https://expressjs.com/en/api.html#req)）。ほとんどの場合、これらのプロパティに手動でアクセスする必要はありません。代わりに、すぐに使える`@Body()`や`@Query()`のような専用のデコレータを使用できます。以下は、提供されているデコレータとそれらが表すプラットフォーム固有のオブジェクトのリストです。

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td></tr>
    <tr>
      <td><code>@Response(), @Res()</code><span class="table-code-asterisk">*</span></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(key?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[key]</code></td>
    </tr>
    <tr>
      <td><code>@Body(key?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[key]</code></td>
    </tr>
    <tr>
      <td><code>@Query(key?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[key]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(name?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[name]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

<sup>\* </sup>基盤となる HTTP プラットフォーム（例：Express や Fastify）間での型定義の互換性のために、Nest は`@Res()`と`@Response()`デコレータを提供しています。`@Res()`は単に`@Response()`のエイリアスです。どちらも基盤となるネイティブプラットフォームの`response`オブジェクトインターフェースを直接公開します。これらを使用する際は、基盤となるライブラリの型定義（例：`@types/express`）もインポートして、完全に活用すべきです。`@Res()`または`@Response()`をメソッドハンドラーに注入すると、そのハンドラーに対して Nest が**ライブラリ固有モード**になり、レスポンスの管理があなたの責任となることに注意してください。その場合、`response`オブジェクトに対して何らかの呼び出し（例：`res.json(...)`や`res.send(...)`）を行ってレスポンスを発行する必要があります。そうしないと、HTTP サーバーがハングします。

> info **ヒント** 独自のカスタムデコレータの作成方法については、[こちら](/custom-decorators)の章を参照してください。

#### リソース

先ほど、cats リソースを取得するエンドポイント（**GET**ルート）を定義しました。通常、新しいレコードを作成するエンドポイントも提供したいと思います。そのために、**POST**ハンドラーを作成しましょう：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create() {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

このように簡単です。Nest は標準的なすべての HTTP メソッド用のデコレータを提供しています：`@Get()`、`@Post()`、`@Put()`、`@Delete()`、`@Patch()`、`@Options()`、`@Head()`。さらに、`@All()`はこれらすべてを処理するエンドポイントを定義します。

#### ルートワイルドカード

NestJS ではパターンベースのルートもサポートされています。例えば、アスタリスク（`*`）をワイルドカードとして使用して、パスの末尾でルート内の任意の文字の組み合わせにマッチさせることができます。次の例では、`findAll()`メソッドは`abcd/`で始まるすべてのルートに対して実行されます。後に続く文字数に関係ありません。

```typescript
@Get('abcd/*')
findAll() {
  return 'This route uses a wildcard';
}
```

`'abcd/*'`ルートパスは、`abcd/`、`abcd/123`、`abcd/abc`などにマッチします。ハイフン（`-`）とドット（`.`）は文字列ベースのパスで文字通りに解釈されます。

このアプローチは Express と Fastify の両方で動作します。ただし、Express の最新リリース（v5）では、ルーティングシステムがより厳密になっています。純粋な Express では、ルートを機能させるために名前付きワイルドカードを使用する必要があります—例えば、`abcd/*splat`です。ここで`splat`は単にワイルドカードパラメータの名前であり、特別な意味はありません。好きな名前を付けることができます。とはいえ、Nest は Express の互換性レイヤーを提供しているため、引き続きアスタリスク（`*`）をワイルドカードとして使用できます。

**ルートの途中**でアスタリスクを使用する場合、Express では名前付きワイルドカード（例：`ab{{ '{' }}*splat&#125;cd`）が必要ですが、Fastify ではまったくサポートされていません。

#### ステータスコード

前述のように、レスポンスのデフォルトの**ステータスコード**は常に**200**です。ただし、POST リクエストの場合は**201**がデフォルトとなります。この動作は、ハンドラーレベルで`@HttpCode(...)`デコレータを使用することで簡単に変更できます。

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> info **ヒント** `HttpCode`は`@nestjs/common`パッケージからインポートしてください。

多くの場合、ステータスコードは静的ではなく、様々な要因に依存します。その場合、ライブラリ固有の**レスポンス**オブジェクト（`@Res()`を使用して注入）を使用するか、エラーの場合は例外をスローすることができます。

#### レスポンスヘッダー

カスタムレスポンスヘッダーを指定するには、`@Header()`デコレータを使用するか、ライブラリ固有のレスポンスオブジェクトを使用して（`res.header()`を直接呼び出して）設定することができます。

```typescript
@Post()
@Header('Cache-Control', 'no-store')
create() {
  return 'This action adds a new cat';
}
```

はい、その部分を翻訳いたします：

> info **ヒント** `Header`は`@nestjs/common`パッケージからインポートしてください。

#### リダイレクト

レスポンスを特定の URL にリダイレクトするには、`@Redirect()`デコレータを使用するか、ライブラリ固有のレスポンスオブジェクトを使用して（`res.redirect()`を直接呼び出して）設定することができます。

`@Redirect()`は 2 つの引数`url`と`statusCode`を取りますが、どちらもオプションです。`statusCode`を省略した場合、デフォルト値は`302`（`Found`）となります。

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> info **ヒント** HTTP ステータスコードやリダイレクト URL を動的に決定したい場合があります。これは`HttpRedirectResponse`インターフェース（`@nestjs/common`から）に従うオブジェクトを返すことで実現できます。

戻り値は`@Redirect()`デコレータに渡された引数を上書きします。例えば：

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

#### ルートパラメータ

静的なパスを持つルートは、リクエストの一部として**動的なデータ**を受け入れる必要がある場合（例：ID `1`の cat を取得するための`GET /cats/1`）には機能しません。パラメータを持つルートを定義するには、URL から動的な値を取得するためのルートパラメータ**トークン**をルートパスに追加できます。以下の`@Get()`デコレータの例では、このアプローチを示すルートパラメータトークンを使用しています。これらのルートパラメータには、メソッドのシグネチャに追加する`@Param()`デコレータを使用してアクセスできます。

> info **ヒント** パラメータを持つルートは、静的なパスの後に宣言する必要があります。これにより、パラメータ化されたパスが静的なパスに向けられたトラフィックを横取りするのを防ぎます。

```typescript
@@filename()
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
@@switch
@Get(':id')
@Bind(Param())
findOne(params) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()`デコレータは、メソッドパラメータを装飾するために使用され（上記の例では`params`）、メソッド内でそのデコレートされたメソッドパラメータのプロパティとして**ルート**パラメータにアクセスできるようにします。コードに示されているように、`params.id`を参照することで`id`パラメータにアクセスできます。あるいは、デコレータに特定のパラメータトークンを渡して、メソッド本体内でルートパラメータを名前で直接参照することもできます。

> info **ヒント** `Param`は`@nestjs/common`パッケージからインポートしてください。

```typescript
@@filename()
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
@@switch
@Get(':id')
@Bind(Param('id'))
findOne(id) {
  return `This action returns a #${id} cat`;
}
```

#### サブドメインルーティング

`@Controller`デコレータは`host`オプションを取ることができ、受信リクエストの HTTP ホストが特定の値と一致することを要求できます。

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> warning **警告** **Fastify**はネストされたルーターをサポートしていないため、サブドメインルーティングを使用する場合は、代わりにデフォルトの Express アダプターを使用することをお勧めします。

ルートの`path`と同様に、`hosts`オプションでもトークンを使用して、ホスト名のその位置にある動的な値を取得することができます。以下の`@Controller()`デコレータの例では、ホストパラメータトークンのこの使用方法を示しています。このように宣言されたホストパラメータには、メソッドのシグネチャに追加する`@HostParam()`デコレータを使用してアクセスできます。

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

#### 状態の共有

他のプログラミング言語から来た開発者にとって、Nest では、ほぼすべてのものが受信リクエスト間で共有されていることを知ると驚くかもしれません。これには、データベース接続プール、グローバル状態を持つシングルトンサービスなどのリソースが含まれます。Node.js は、各リクエストが別々のスレッドによって処理されるリクエスト/レスポンスのマルチスレッド・ステートレスモデルを使用していないことを理解することが重要です。その結果、Nest でシングルトンインスタンスを使用することは、アプリケーションにとって完全に**安全**です。

とはいえ、コントローラーにリクエストベースのライフタイムが必要となる特定のエッジケースが存在します。例として、GraphQL アプリケーションでのリクエストごとのキャッシング、リクエストの追跡、マルチテナンシーの実装などがあります。インジェクションスコープの制御については[こちら](/fundamentals/injection-scopes)で詳しく学ぶことができます。

#### 非同期性

私たちは現代の JavaScript、特に**非同期**データ処理の重視を愛しています。そのため、Nest は`async`関数を完全にサポートしています。すべての`async`関数は`Promise`を返す必要があり、これにより Nest が自動的に解決できる遅延値を返すことができます。以下は例です：

```typescript
@@filename(cats.controller)
@Get()
async findAll(): Promise<any[]> {
  return [];
}
@@switch
@Get()
async findAll() {
  return [];
}
```

このコードは完全に有効です。しかし、Nest はさらに一歩進んで、ルートハンドラーが RxJS の[observable ストリーム](https://rxjs-dev.firebaseapp.com/guide/observable)を返すことも可能にしています。Nest は内部でサブスクリプションを処理し、ストリームが完了すると最終的に発行された値を解決します。

```typescript
@@filename(cats.controller)
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
@@switch
@Get()
findAll() {
  return of([]);
}
```

両方のアプローチが有効で、ニーズに最も適したものを選択できます。

#### リクエストペイロード

前回の例では、POST ルートハンドラーはクライアントパラメータを受け付けていませんでした。`@Body()`デコレータを追加して、これを修正しましょう。

先に進む前に（TypeScript を使用している場合）、**DTO**（Data Transfer Object）スキーマを定義する必要があります。DTO は、データがネットワーク上でどのように送信されるべきかを指定するオブジェクトです。DTO スキーマは**TypeScript**のインターフェースや単純なクラスを使用して定義できます。しかし、ここでは**クラス**の使用を推奨します。なぜでしょうか？クラスは JavaScript ES6 標準の一部であり、コンパイルされた JavaScript でも実体として残ります。対照的に、TypeScript のインターフェースはトランスパイル時に削除され、Nest は実行時にそれらを参照できなくなります。これは重要です。なぜなら、**Pipes**のような機能は実行時に変数のメタタイプにアクセスする必要があり、これはクラスでのみ可能だからです。

`CreateCatDto`クラスを作成してみましょう：

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

これには 3 つの基本的なプロパティしかありません。その後、新しく作成した DTO を`CatsController`内で使用できます：

```typescript
@@filename(cats.controller)
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
@@switch
@Post()
@Bind(Body())
async create(createCatDto) {
  return 'This action adds a new cat';
}
```

> info **ヒント** 私たちの`ValidationPipe`は、メソッドハンドラーが受け取るべきでないプロパティをフィルタリングできます。この場合、受け入れ可能なプロパティをホワイトリストに登録でき、ホワイトリストに含まれていないプロパティは結果のオブジェクトから自動的に除外されます。`CreateCatDto`の例では、ホワイトリストは`name`、`age`、`breed`プロパティです。詳しくは[こちら](https://docs.nestjs.com/techniques/validation#stripping-properties)をご覧ください。

#### クエリパラメータ

ルートでクエリパラメータを処理する際、`@Query()`デコレータを使用して受信リクエストからそれらを抽出できます。実際にどのように動作するか見てみましょう。

例えば、`age`や`breed`のようなクエリパラメータに基づいて cats のリストをフィルタリングしたい場合を考えてみましょう。まず、`CatsController`でクエリパラメータを定義します：

```typescript
@@filename(cats.controller)
@Get()
async findAll(@Query('age') age: number, @Query('breed') breed: string) {
  return `This action returns all cats filtered by age: ${age} and breed: ${breed}`;
}
```

この例では、`@Query()`デコレータを使用してクエリ文字列から`age`と`breed`の値を抽出しています。例えば、次のようなリクエストの場合：

```plaintext
GET /cats?age=2&breed=Persian
```

`age`は`2`、`breed`は`Persian`となります。

アプリケーションでネストされたオブジェクトや配列のような、より複雑なクエリパラメータを処理する必要がある場合：

```plaintext
?filter[where][name]=John&filter[where][age]=30
?item[]=1&item[]=2
```

HTTP アダプター（Express または Fastify）で適切なクエリパーサーを使用するように設定する必要があります。Express では、リッチなクエリオブジェクトを可能にする`extended`パーサーを使用できます：

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
app.set('query parser', 'extended');
```

Fastify では、`querystringParser`オプションを使用できます：

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({
    querystringParser: (str) => qs.parse(str),
  }),
);
```

> info **ヒント** `qs`はネストと配列をサポートするクエリ文字列パーサーです。`npm install qs`を使用してインストールできます。

はい、その部分を翻訳いたします：

#### エラー処理

エラー処理（つまり、例外の処理）については、[こちら](/exception-filters)に別の章があります。

#### 完全なリソースサンプル

以下は、基本的なコントローラーを作成するために利用可能な複数のデコレータの使用方法を示す例です。このコントローラーは、内部データにアクセスして操作するためのいくつかのメソッドを提供します。

```typescript
@@filename(cats.controller)
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
@@switch
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat`;
  }
}
```

> info **ヒント** Nest CLI は、**すべてのボイラープレートコード**を自動的に作成するジェネレーター（スキーマ）を提供しており、手動での作業を省き、全体的な開発者体験を向上させます。この機能の詳細については[こちら](/recipes/crud-generator)をご覧ください。

#### 起動と実行

`CatsController`が完全に定義されていても、Nest はまだそれを認識しておらず、クラスのインスタンスを自動的に作成することはありません。

コントローラーは常にモジュールの一部である必要があります。そのため、`@Module()`デコレータ内に`controllers`配列を含めています。ルートの`AppModule`以外のモジュールを定義していないため、`CatsController`を登録するためにこれを使用します：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

`@Module()`デコレータを使用してモジュールクラスにメタデータを付加し、これにより Nest はどのコントローラーをマウントする必要があるかを簡単に判断できるようになります。

はい、その部分を翻訳いたします：

#### ライブラリ固有のアプローチ

ここまで、レスポンスを操作する標準的な Nest の方法について説明してきました。もう 1 つのアプローチは、ライブラリ固有の[レスポンスオブジェクト](https://expressjs.com/en/api.html#res)を使用することです。特定のレスポンスオブジェクトを注入するには、`@Res()`デコレータを使用できます。違いを明確にするために、`CatsController`を次のように書き換えてみましょう：

```typescript
@@filename()
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
@@switch
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

このアプローチは機能し、レスポンスオブジェクトへの完全な制御（ヘッダーの操作やライブラリ固有の機能へのアクセスなど）を提供することでより柔軟性を提供しますが、注意して使用する必要があります。一般的に、このメソッドはあまり明確ではなく、いくつかの欠点があります。主な欠点は、異なる基盤となるライブラリがレスポンスオブジェクトに対して異なる API を持つ可能性があるため、コードがプラットフォーム依存になることです。さらに、レスポンスオブジェクトのモックを作成する必要があるなど、テストがより困難になる可能性があります。

さらに、このアプローチを使用すると、インターセプターや`@HttpCode()` / `@Header()`デコレータなど、標準的なレスポンス処理に依存する Nest の機能との互換性が失われます。これに対処するには、次のように`passthrough`オプションを有効にできます：

```typescript
@@filename()
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
@@switch
@Get()
@Bind(Res({ passthrough: true }))
findAll(res) {
  res.status(HttpStatus.OK);
  return [];
}
```

このアプローチを使用すると、ネイティブのレスポンスオブジェクトと対話できます（例えば、特定の条件に基づいてクッキーやヘッダーを設定するなど）が、残りの処理はフレームワークに任せることができます。
