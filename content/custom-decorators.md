### カスタムルートデコレーター

Nestは**デコレーター**と呼ばれる言語機能を中心に構築されています。デコレーターは多くの一般的なプログラミング言語でよく知られている概念ですが、JavaScriptの世界ではまだ比較的新しいものです。デコレーターの仕組みをより良く理解するために、[この記事](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)を読むことをお勧めします。以下は簡単な定義です：

<blockquote class="external">
  ES2016デコレーターは関数を返す式で、ターゲット、名前、プロパティディスクリプタを引数として取ることができます。
  デコレーターの先頭に<code>@</code>文字を付けて、装飾したいものの一番上に配置することで適用します。デコレーターはクラス、メソッド、またはプロパティに対して定義できます。
</blockquote>

#### パラメーターデコレーター

Nestは、HTTPルートハンドラーと一緒に使用できる便利な**パラメーターデコレーター**のセットを提供します。以下は提供されているデコレーターとそれらが表すプレーンなExpress（またはFastify）オブジェクトのリストです：

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code></td>
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
      <td><code>@Param(param?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[param]</code></td>
    </tr>
    <tr>
      <td><code>@Body(param?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[param]</code></td>
    </tr>
    <tr>
      <td><code>@Query(param?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[param]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(param?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[param]</code></td>
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

さらに、独自の**カスタムデコレーター**を作成することもできます。なぜこれが有用なのでしょうか？

Node.jsの世界では、**request**オブジェクトにプロパティを付加することが一般的な慣行です。その後、各ルートハンドラーで以下のようなコードを使用して手動で抽出します：

```typescript
const user = req.user;
```

コードをより読みやすく透明にするために、`@User()`デコレーターを作成し、すべてのコントローラーで再利用することができます：

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

そして、必要に応じて簡単に使用することができます：

```typescript
@@filename()
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
@@switch
@Get()
@Bind(User())
async findOne(user) {
  console.log(user);
}
```

#### データの受け渡し

デコレーターの動作が特定の条件に依存する場合、`data`パラメーターを使用してデコレーターのファクトリー関数に引数を渡すことができます。これの一つのユースケースは、キーによってリクエストオブジェクトからプロパティを抽出するカスタムデコレーターです。例えば、<a href="techniques/authentication#implementing-passport-strategies">認証レイヤー</a>がリクエストを検証し、ユーザーエンティティをリクエストオブジェクトに付加すると仮定します。認証されたリクエストのユーザーエンティティは以下のようになります：

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

プロパティ名をキーとして受け取り、関連する値が存在する場合はその値を返す（存在しない場合や`user`オブジェクトが作成されていない場合はundefined）デコレーターを定義してみましょう：

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
@@switch
import { createParamDecorator } from '@nestjs/common';

export const User = createParamDecorator((data, ctx) => {
  const request = ctx.switchToHttp().getRequest();
  const user = request.user;

  return data ? user && user[data] : user;
});
```

コントローラーで`@User()`デコレーターを使用して特定のプロパティにアクセスする方法は以下の通りです：

```typescript
@@filename()
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
@@switch
@Get()
@Bind(User('firstName'))
async findOne(firstName) {
  console.log(`Hello ${firstName}`);
}
```

このデコレーターを異なるキーで使用して、異なるプロパティにアクセスすることができます。`user`オブジェクトが深いまたは複雑な場合、これによってリクエストハンドラーの実装がより簡単で読みやすくなります。

> info **ヒント** TypeScriptユーザーの場合、`createParamDecorator<T>()`はジェネリックであることに注意してください。これは型安全性を明示的に強制できることを意味します。例えば`createParamDecorator<string>((data, ctx) => ...)`のように書くことができます。または、ファクトリー関数でパラメーター型を指定することもできます。例：`createParamDecorator((data: string, ctx) => ...)`。両方を省略した場合、`data`の型は`any`になります。

#### パイプの使用

Nestはカスタムパラメーターデコレーターを組み込みのもの（`@Body()`、`@Param()`、`@Query()`）と同じように扱います。これは、カスタムデコレーターが付けられたパラメーター（例では`user`引数）に対してもパイプが実行されることを意味します。さらに、パイプをカスタムデコレーターに直接適用することもできます：

```typescript
@@filename()
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
@@switch
@Get()
@Bind(User(new ValidationPipe({ validateCustomDecorators: true })))
async findOne(user) {
  console.log(user);
}
```

> info **ヒント** `validateCustomDecorators`オプションをtrueに設定する必要があることに注意してください。`ValidationPipe`はデフォルトではカスタムデコレーターで注釈付けされた引数を検証しません。

#### デコレーターの合成

Nestは複数のデコレーターを合成するためのヘルパーメソッドを提供します。例えば、認証に関連するすべてのデコレーターを1つのデコレーターにまとめたい場合、以下のような構成で実現できます：

```typescript
@@filename(auth.decorator)
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
@@switch
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

このカスタム`@Auth()`デコレーターは以下のように使用できます：

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

これにより、4つのデコレーターすべてを1つの宣言で適用する効果があります。

> warning **警告** `@nestjs/swagger`パッケージの`@ApiHideProperty()`デコレーターは合成可能ではなく、`applyDecorators`関数では正しく動作しません。

カスタムデコレーターの主要なユースケースをまとめます：

1. ユーザー認証/認可関連
```typescript
@Auth('admin')  // 役割ベースの認証
@RequirePermissions(['read', 'write'])  // 権限チェック
@IsAuthenticated()  // 認証状態の確認
```

2. リクエストデータの抽出と加工
```typescript
@User()  // ユーザー情報の抽出
@Company()  // 企業情報の抽出
@CurrentTenant()  // マルチテナント環境でのテナント情報取得
```

3. バリデーションと変換
```typescript
@Sanitize()  // 入力データのサニタイズ
@Transform(TransformationType.ToNumber)  // データ型変換
@ValidateIf(condition)  // 条件付きバリデーション
```

4. キャッシュと最適化
```typescript
@Cache(TTL.FIVE_MINUTES)  // レスポンスのキャッシュ
@NoCache()  // キャッシュの無効化
@CompressResponse()  // レスポンス圧縮
```

5. ロギングとモニタリング
```typescript
@Log('api-access')  // アクセスログ
@TrackPerformance()  // パフォーマンス測定
@Metric('endpoint-calls')  // メトリクス収集
```

6. 複数のデコレーターの組み合わせ
```typescript
@ApiEndpoint({ 
  roles: ['admin'], 
  cache: true,
  log: true 
})  // 複数の機能を1つのデコレーターにまとめる
```

これらのカスタムデコレーターを使用することで：
- コードの重複を減らせる
- 横断的な関心事を分離できる
- 宣言的なプログラミングスタイルを実現できる
- テストがしやすくなる
- コードの可読性が向上する

実装の際は、デコレーターが単一責任の原則に従い、再利用可能で、適切にテストされていることを確認することが重要です。
