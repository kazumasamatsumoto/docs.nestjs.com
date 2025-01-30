### ガード

ガードは`@Injectable()`デコレータで装飾され、`CanActivate`インターフェースを実装したクラスです。

ガードは**単一の責任**を持ちます。実行時に存在する特定の条件（権限、ロール、ACLなど）に応じて、与えられたリクエストをルートハンドラで処理するかどうかを決定します。これは一般的に**認可**と呼ばれます。認可（およびそれと密接に関連する**認証**）は、従来のExpressアプリケーションでは[ミドルウェア](/middleware)によって処理されてきました。トークンの検証やリクエストオブジェクトへのプロパティの付加は特定のルートコンテキスト（およびそのメタデータ）と強く結びついていないため、ミドルウェアは認証には適しています。

しかし、ミドルウェアは本質的に単純です。`next()`関数を呼び出した後にどのハンドラが実行されるかを知りません。一方、**ガード**は`ExecutionContext`インスタンスにアクセスでき、次に何が実行されるかを正確に把握しています。例外フィルター、パイプ、インターセプターと同様に、リクエスト/レスポンスサイクルの適切なポイントで処理ロジックを宣言的に挿入できるように設計されています。これによりコードをDRYで宣言的に保つことができます。

> info **ヒント** ガードは全てのミドルウェアの**後**、かつインターセプターやパイプの**前**に実行されます。

#### 認可ガード

前述の通り、**認可**はガードの優れたユースケースです。特定のルートは、呼び出し元（通常は特定の認証されたユーザー）が十分な権限を持っている場合にのみ利用可能であるべきだからです。これから作成する`AuthGuard`は、認証済みのユーザー（したがってトークンがリクエストヘッダーに添付されている）を前提としています。トークンを抽出して検証し、抽出された情報を使用してリクエストを進めることができるかどうかを判断します。

```typescript
@@filename(auth.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class AuthGuard {
  async canActivate(context) {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> info **ヒント** アプリケーションで認証メカニズムを実装する実世界の例をお探しの場合は、[この章](/security/authentication)をご覧ください。同様に、より洗練された認可の例については、[このページ](/security/authorization)をチェックしてください。

`validateRequest()`関数内のロジックは、必要に応じて単純にも複雑にもできます。この例の主なポイントは、ガードがリクエスト/レスポンスサイクルにどのように適合するかを示すことです。

全てのガードは`canActivate()`関数を実装する必要があります。この関数は、現在のリクエストが許可されるかどうかを示すブール値を返す必要があります。同期的または非同期的（`Promise`や`Observable`を介して）にレスポンスを返すことができます。Nestは戻り値を使用して次のアクションを制御します：

- `true`を返した場合、リクエストは処理されます。
- `false`を返した場合、Nestはリクエストを拒否します。

#### 実行コンテキスト

`canActivate()`関数は、`ExecutionContext`インスタンスを単一の引数として取ります。`ExecutionContext`は`ArgumentsHost`を継承します。`ArgumentsHost`については、以前に例外フィルターの章で見ました。上記のサンプルでは、以前使用した`ArgumentsHost`で定義された同じヘルパーメソッドを使用して、`Request`オブジェクトへの参照を取得しています。このトピックの詳細については、例外フィルターの章の**Arguments host**セクションを参照してください。

`ArgumentsHost`を拡張することで、`ExecutionContext`は現在の実行プロセスに関する追加の詳細を提供する新しいヘルパーメソッドをいくつか追加します。これらの詳細は、広範なコントローラー、メソッド、実行コンテキストで動作できるより汎用的なガードを構築する際に役立ちます。`ExecutionContext`の詳細については[こちら](/fundamentals/execution-context)をご覧ください。

#### ロールベースの認証

特定のロールを持つユーザーのみにアクセスを許可する、より機能的なガードを構築しましょう。基本的なガードのテンプレートから始めて、以降のセクションで拡張していきます。現時点では、全てのリクエストの処理を許可します：

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  canActivate(context) {
    return true;
  }
}
```

#### ガードのバインディング

パイプや例外フィルターと同様に、ガードも**コントローラースコープ**、メソッドスコープ、またはグローバルスコープにすることができます。以下では、`@UseGuards()`デコレータを使用してコントローラースコープのガードを設定します。このデコレータは単一の引数、またはカンマで区切られた引数のリストを取ることができます。これにより、1つの宣言で適切なガードのセットを簡単に適用できます。

```typescript
@@filename()
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> info **ヒント** `@UseGuards()`デコレータは`@nestjs/common`パッケージからインポートされます。

上記では、`RolesGuard`クラスをインスタンス化の責任をフレームワークに任せ、依存性の注入を可能にするために、インスタンスではなくクラスを渡しました。パイプや例外フィルターと同様に、インスタンスをその場で渡すこともできます：

```typescript
@@filename()
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

上記の構成は、このコントローラーで宣言された全てのハンドラーにガードを適用します。ガードを単一のメソッドにのみ適用したい場合は、`@UseGuards()`デコレータを**メソッドレベル**で適用します。

グローバルガードを設定するには、Nestアプリケーションインスタンスの`useGlobalGuards()`メソッドを使用します：

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> warning **注意** ハイブリッドアプリケーションの場合、`useGlobalGuards()`メソッドはデフォルトではゲートウェイやマイクロサービスに対してガードを設定しません（この動作を変更する方法については[ハイブリッドアプリケーション](/faq/hybrid-application)を参照してください）。「標準的な」（非ハイブリッドの）マイクロサービスアプリケーションでは、`useGlobalGuards()`はガードをグローバルにマウントします。

グローバルガードは、全てのコントローラーと全てのルートハンドラーに対して、アプリケーション全体で使用されます。依存性の注入に関して、モジュールの外部から登録されたグローバルガード（上記の例のように`useGlobalGuards()`を使用）は、これがモジュールのコンテキスト外で行われるため、依存性を注入することができません。この問題を解決するには、以下の構成を使用して任意のモジュールから直接ガードを設定することができます：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> info **ヒント** このアプローチを使用してガードの依存性注入を実行する場合、この構成が採用されるモジュールに関係なく、ガードは実際にはグローバルであることに注意してください。これはどこで行うべきでしょうか？ガード（上記の例では`RolesGuard`）が定義されているモジュールを選択してください。また、`useClass`はカスタムプロバイダーの登録を扱う唯一の方法ではありません。詳細は[こちら](/fundamentals/custom-providers)をご覧ください。

#### ハンドラーごとのロールの設定

`RolesGuard`は動作していますが、まだあまりスマートではありません。ガードの最も重要な機能である[実行コンテキスト](/fundamentals/execution-context)をまだ活用できていません。ロールについて、また各ハンドラーにどのロールが許可されているかについてまだ把握していません。例えば、`CatsController`は異なるルートに対して異なる権限スキームを持つことができます。一部は管理者ユーザーのみが利用可能で、他は誰でも利用可能かもしれません。柔軟で再利用可能な方法でロールをルートに一致させるにはどうすればよいでしょうか？

ここで**カスタムメタデータ**が重要になってきます（詳細は[こちら](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)をご覧ください）。Nestは、`Reflector#createDecorator`静的メソッドを使用して作成されたデコレータ、または組み込みの`@SetMetadata()`デコレータを通じて、ルートハンドラーにカスタム**メタデータ**を付加する機能を提供します。

例えば、`Reflector#createDecorator`メソッドを使用して、メタデータをハンドラーに付加する`@Roles()`デコレータを作成してみましょう。`Reflector`はフレームワークによってあらかじめ提供されており、`@nestjs/core`パッケージから公開されています。

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

ここでの`Roles`デコレータは、`string[]`型の単一の引数を取る関数です。

このデコレータを使用するには、単にハンドラーに注釈を付けます：

```typescript
@@filename(cats.controller)
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

ここでは、`create()`メソッドに`Roles`デコレータメタデータを付加し、`admin`ロールを持つユーザーのみがこのルートにアクセスできることを示しています。

あるいは、`Reflector#createDecorator`メソッドを使用する代わりに、組み込みの`@SetMetadata()`デコレータを使用することもできます。詳細は[こちら](/fundamentals/execution-context#low-level-approach)をご覧ください。

#### 全てを組み合わせる

では、これを`RolesGuard`と組み合わせてみましょう。現在は全てのケースで単に`true`を返し、全てのリクエストを処理できるようにしています。**現在のユーザーに割り当てられたロール**と、処理中の現在のルートに必要な実際のロールを比較して、戻り値を条件付きにしたいと思います。ルートのロール（カスタムメタデータ）にアクセスするために、再び`Reflector`ヘルパークラスを使用します：

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

```typescript
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> info **ヒント** Node.jsの世界では、認可されたユーザーを`request`オブジェクトに付加するのが一般的な慣行です。したがって、上記のサンプルコードでは、`request.user`にユーザーインスタンスと許可されたロールが含まれていることを前提としています。あなたのアプリケーションでは、おそらくカスタムの**認証ガード**（またはミドルウェア）でその関連付けを行うことになるでしょう。このトピックの詳細については[この章](/security/authentication)をチェックしてください。

> warning **警告** `matchRoles()`関数内のロジックは、必要に応じて単純にも複雑にもできます。この例の主なポイントは、ガードがリクエスト/レスポンスサイクルにどのように適合するかを示すことです。

コンテキストに応じた`Reflector`の利用の詳細については、**実行コンテキスト**の章の<a href="https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata">リフレクションとメタデータ</a>セクションを参照してください。

不十分な権限を持つユーザーがエンドポイントをリクエストした場合、Nestは自動的に以下のレスポンスを返します：

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

なお、裏側では、ガードが`false`を返した場合、フレームワークは`ForbiddenException`をスローします。異なるエラーレスポンスを返したい場合は、独自の特定の例外をスローする必要があります。例えば：

```typescript
throw new UnauthorizedException();
```

ガードによってスローされた例外は、[例外レイヤー](/exception-filters)（グローバル例外フィルターと現在のコンテキストに適用される例外フィルター）によって処理されます。

> info **ヒント** 認可を実装する実世界の例をお探しの場合は、[この章](/security/authorization)をチェックしてください。

ガードの一般的なユースケースをご紹介します：

1. 認証（Authentication）
```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    // JWTトークンの検証
    return validateAuthToken(request.headers.authorization);
  }
}
```
- ユーザーが正当な認証トークンを持っているか確認
- 未認証ユーザーのアクセスを制限

2. ロールベースの認可（Role-based Authorization）
```typescript
@Injectable()
export class AdminGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return user && user.role === 'admin';
  }
}
```
- 管理者専用のエンドポイントの保護
- 特定のロールを持つユーザーのみアクセス可能

3. APIレート制限（Rate Limiting）
```typescript
@Injectable()
export class ThrottlerGuard implements CanActivate {
  constructor(private readonly limiter: RateLimiterService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const clientIp = request.ip;
    return this.limiter.checkRate(clientIp);
  }
}
```
- APIの呼び出し回数を制限
- DDoS攻撃の防止

4. リソースアクセス制御（Resource Access Control）
```typescript
@Injectable()
export class ResourceOwnerGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resourceId = request.params.id;
    
    return this.checkResourceOwnership(user.id, resourceId);
  }
}
```
- ユーザーが自分のリソースのみにアクセスできることを保証
- 他のユーザーのデータへの不正アクセスを防止

5. 環境ベースの制限（Environment-based Restrictions）
```typescript
@Injectable()
export class ProductionGuard implements CanActivate {
  canActivate(): boolean {
    return process.env.NODE_ENV !== 'production';
  }
}
```
- 特定の環境でのみ利用可能なエンドポイントの制御
- 開発用エンドポイントの本番環境での無効化

6. 時間ベースのアクセス制御（Time-based Access Control）
```typescript
@Injectable()
export class BusinessHoursGuard implements CanActivate {
  canActivate(): boolean {
    const currentHour = new Date().getHours();
    return currentHour >= 9 && currentHour < 17; // 9AM-5PM
  }
}
```
- 特定の時間帯のみアクセス可能
- メンテナンス時間中のアクセス制限

これらのガードは単独で使用することも、`@UseGuards()`デコレータを使用して組み合わせることも可能です：

```typescript
@Controller('admin')
@UseGuards(AuthGuard, AdminGuard, BusinessHoursGuard)
export class AdminController {
  // 認証済み、管理者権限持ち、営業時間内のユーザーのみアクセス可能
}
```

これらのユースケースは、アプリケーションのセキュリティと制御を強化するための基本的な実装例です。実際のアプリケーションでは、これらを基にして、より詳細なビジネスロジックや要件に合わせてカスタマイズすることができます。
