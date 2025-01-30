### インターセプター

インターセプターは`@Injectable()`デコレーターで修飾され、`NestInterceptor`インターフェースを実装したクラスです。

<figure><img class="illustrative-image" src="/assets/Interceptors_1.png" /></figure>

インターセプターは[アスペクト指向プログラミング](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP)の手法にインスパイアされた、以下のような便利な機能を提供します：

- メソッド実行の前後に追加のロジックをバインドする
- 関数から返される結果を変換する
- 関数からスローされた例外を変換する
- 基本的な関数の動作を拡張する
- 特定の条件に応じて関数を完全にオーバーライドする(例：キャッシュ目的)

#### 基本

各インターセプターは2つの引数を取る`intercept()`メソッドを実装します。1つ目は`ExecutionContext`インスタンス([ガード](/guards)と同じオブジェクト)です。`ExecutionContext`は`ArgumentsHost`を継承しています。`ArgumentsHost`については例外フィルターの章で見ました。そこで見たように、これは元のハンドラーに渡された引数のラッパーであり、アプリケーションの種類に応じて異なる引数配列を含んでいます。この詳細については[例外フィルター](https://docs.nestjs.com/exception-filters#arguments-host)を参照してください。

#### 実行コンテキスト

`ExecutionContext`は`ArgumentsHost`を拡張することで、現在の実行プロセスに関する追加の詳細を提供するヘルパーメソッドをいくつか追加します。これらの詳細は、より汎用的なインターセプターを構築する際に役立ちます。`ExecutionContext`の詳細については[こちら](/fundamentals/execution-context)をご覧ください。

#### コールハンドラー 

2つ目の引数は`CallHandler`です。`CallHandler`インターフェースは`handle()`メソッドを実装しており、これを使用してインターセプター内の任意の時点でルートハンドラーメソッドを呼び出すことができます。`intercept()`メソッドの実装で`handle()`メソッドを呼び出さない場合、ルートハンドラーメソッドは一切実行されません。

この方式により、`intercept()`メソッドはリクエスト/レスポンスのストリームを効果的に**ラップ**します。その結果、最終的なルートハンドラーの実行の**前後**に独自のロジックを実装できます。`handle()`を呼び出す**前**にコードを実行できるのは明らかですが、その後の動作にはどのように影響を与えられるのでしょうか？`handle()`メソッドは`Observable`を返すため、強力な[RxJS](https://github.com/ReactiveX/rxjs)オペレーターを使用してレスポンスをさらに操作できます。アスペクト指向プログラミングの用語では、ルートハンドラーの呼び出し(つまり`handle()`の呼び出し)は[ポイントカット](https://en.wikipedia.org/wiki/Pointcut)と呼ばれ、追加のロジックが挿入されるポイントを示します。

例えば、incoming `POST /cats`リクエストを考えてみましょう。このリクエストは`CatsController`内で定義された`create()`ハンドラーに向かいます。`handle()`メソッドを呼び出さないインターセプターが途中で呼び出された場合、`create()`メソッドは実行されません。`handle()`が呼び出され(`Observable`が返された後)、`create()`ハンドラーがトリガーされます。そして`Observable`を介してレスポンスストリームを受信すると、ストリームに追加の操作を実行し、最終結果を呼び出し元に返すことができます。

#### アスペクトインターセプション

最初のユースケースとして、ユーザーの操作をログに記録する(例：ユーザーの呼び出しを保存、イベントを非同期にディスパッチ、タイムスタンプを計算する)ためにインターセプターを使用する例を見てみましょう。以下に簡単な`LoggingInterceptor`を示します：

```typescript
@@filename(logging.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

> info **ヒント** `NestInterceptor<T, R>`は汎用インターフェースで、`T`は`Observable<T>`の型(レスポンスストリームをサポート)を示し、`R`は`Observable<R>`でラップされた値の型です。

> warning **注意** インターセプターは、コントローラーやプロバイダー、ガードなどと同様に、`constructor`を通じて依存関係を**注入**できます。

`handle()`はRxJSの`Observable`を返すため、ストリームを操作するための幅広いオペレーターを選択できます。上記の例では、`tap()`オペレーターを使用しています。これは、observableストリームの正常または例外的な終了時に匿名のログ関数を呼び出しますが、レスポンスサイクルには干渉しません。

#### インターセプターのバインド

インターセプターを設定するには、`@nestjs/common`パッケージからインポートした`@UseInterceptors()`デコレーターを使用します。[パイプ](/pipes)や[ガード](/guards)と同様に、インターセプターはコントローラースコープ、メソッドスコープ、またはグローバルスコープで設定できます。

```typescript
@@filename(cats.controller)
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> info **ヒント** `@UseInterceptors()`デコレーターは`@nestjs/common`パッケージからインポートされます。

上記の構成を使用すると、`CatsController`で定義された各ルートハンドラーは`LoggingInterceptor`を使用します。誰かが`GET /cats`エンドポイントを呼び出すと、標準出力に以下のような出力が表示されます：

```typescript
Before...
After... 1ms
```

パイプ、ガード、例外フィルターと同様に、インスタンスの代わりに`LoggingInterceptor`クラスを渡し、インスタンス化の責任をフレームワークに任せ、依存性注入を可能にしています。インスタンスをその場で渡すこともできます：

```typescript
@@filename(cats.controller)
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

前述のように、上記の構成はこのコントローラーで宣言された全てのハンドラーにインターセプターを適用します。インターセプターのスコープを単一のメソッドに制限したい場合は、デコレーターを**メソッドレベル**で適用するだけです。

グローバルインターセプターを設定するには、Nestアプリケーションインスタンスの`useGlobalInterceptors()`メソッドを使用します：

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

グローバルインターセプターは、全てのコントローラーと全てのルートハンドラーに対して、アプリケーション全体で使用されます。依存性注入に関して、任意のモジュールの外部で登録されたグローバルインターセプター(上記の例のように`useGlobalInterceptors()`で)は、これが任意のモジュールのコンテキスト外で行われるため、依存関係を注入できません。この問題を解決するには、以下の構成を使用して、インターセプターを**任意のモジュールから直接**設定できます：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> info **ヒント** このアプローチを使用してインターセプターの依存性注入を実行する場合、この構成が使用されるモジュールに関係なく、インターセプターは実際にはグローバルであることに注意してください。これはどこで行うべきでしょうか？インターセプター(上記の例では`LoggingInterceptor`)が定義されているモジュールを選択してください。また、`useClass`はカスタムプロバイダーの登録を扱う唯一の方法ではありません。詳細は[こちら](/fundamentals/custom-providers)をご覧ください。

#### レスポンスマッピング

`handle()`が`Observable`を返すことは既に知っています。ストリームにはルートハンドラーから**返された**値が含まれているため、RxJSの`map()`オペレーターを使用して簡単に変更できます。

> warning **警告** レスポンスマッピング機能は、ライブラリ固有のレスポンス戦略(`@Res()`オブジェクトを直接使用することは禁止されています)では動作しません。

プロセスを示すために、各レスポンスを些細な方法で変更する`TransformInterceptor`を作成してみましょう。RxJSの`map()`オペレーターを使用して、レスポンスオブジェクトを新しく作成されたオブジェクトの`data`プロパティに割り当て、新しいオブジェクトをクライアントに返します。

```typescript
@@filename(transform.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> info **ヒント** Nestインターセプターは同期と非同期の両方の`intercept()`メソッドで動作します。必要に応じて、メソッドを`async`に切り替えるだけです。

上記の構成で、誰かが`GET /cats`エンドポイントを呼び出すと、レスポンスは以下のようになります(ルートハンドラーが空の配列`[]`を返すと仮定):

```json
{
  "data": []
}
```

インターセプターは、アプリケーション全体で発生する要件に再利用可能なソリューションを作成する上で大きな価値があります。
例えば、全ての`null`値を空文字列`''`に変換する必要がある場合を想像してください。1行のコードでそれを行い、インターセプターをグローバルにバインドすることで、登録された各ハンドラーで自動的に使用されるようになります。

```typescript
@@filename()
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

#### 例外マッピング

もう1つの興味深いユースケースは、RxJSの`catchError()`オペレーターを活用してスローされた例外をオーバーライドすることです：

```typescript
@@filename(errors.interceptor)
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
@@switch
import { Injectable, BadGatewayException } from '@nestjs/common';
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor {
  intercept(context, next) {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

#### ストリームのオーバーライド

場合によっては、ハンドラーの呼び出しを完全に防ぎ、代わりに異なる値を返したい理由がいくつかあります。明白な例として、レスポンス時間を改善するためのキャッシュの実装があります。簡単な**キャッシュインターセプター**を見てみましょう。現実的な例では、TTL、キャッシュの無効化、キャッシュサイズなどの他の要因も考慮する必要がありますが、それはこの議論の範囲を超えています。ここでは主要な概念を示す基本的な例を提供します：

```typescript
@@filename(cache.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor {
  intercept(context, next) {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

この`CacheInterceptor`には、ハードコードされた`isCached`変数とハードコードされたレスポンス`[]`があります。重要なポイントは、RxJSの`of()`オペレーターによって作成された新しいストリームを返すことです。そのため、ルートハンドラーは**一切呼び出されません**。`CacheInterceptor`を使用するエンドポイントを誰かが呼び出すと、レスポンス(ハードコードされた空の配列)が即座に返されます。汎用的なソリューションを作成するには、`Reflector`を活用してカスタムデコレーターを作成できます。`Reflector`については[ガード](/guards)の章で詳しく説明されています。

#### その他のオペレーター

RxJSオペレーターを使用してストリームを操作する可能性により、多くの機能が提供されます。別の一般的なユースケースを考えてみましょう。ルートリクエストの**タイムアウト**を処理したいとします。エンドポイントが一定時間後に何も返さない場合、エラーレスポンスで終了させたいとします。以下の構成でこれが可能になります：

```typescript
@@filename(timeout.interceptor)
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
@@switch
import { Injectable, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

5秒後にリクエスト処理がキャンセルされます。`RequestTimeoutException`をスローする前にカスタムロジック(例：リソースの解放)を追加することもできます。

インターセプターの主要なユースケースをご紹介します：

1. ロギングとモニタリング
- リクエスト/レスポンスのログ記録
- 実行時間の計測（パフォーマンスモニタリング）
- アクセスログの記録
- エラーのログ記録と通知

2. キャッシュ
- レスポンスのキャッシュ
- 頻繁にアクセスされるデータのキャッシュ
- 条件付きキャッシュの実装

3. レスポンス変換
- レスポンス形式の標準化（共通のレスポンス構造への変換）
- nullの除外や特定の値の変換
- 機密データの除去やマスキング
- レスポンスの圧縮

4. エラーハンドリング
- 共通のエラー処理
- 特定の例外の変換
- エラーレスポンスの標準化

5. 認証・認可
- トークンの検証
- ユーザーセッションの確認
- アクセス権限の検証
- レート制限の実装

6. トランザクション管理
- データベーストランザクションの開始/コミット
- トランザクションのロールバック処理
- 複数のデータベース操作の一貫性保証

7. 非機能要件の実装
- タイムアウト処理
- リトライ処理
- サーキットブレーカーパターンの実装
- スロットリング

実際の例を示すと：

```typescript
// 認証トークン検証の例
@Injectable()
export class AuthInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const token = request.headers['authorization'];
    
    if (!token) {
      throw new UnauthorizedException();
    }
    
    // トークン検証ロジック
    try {
      request.user = verifyToken(token);
      return next.handle();
    } catch (error) {
      throw new UnauthorizedException();
    }
  }
}

// レスポンス標準化の例
@Injectable()
export class ResponseFormatInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
        path: context.switchToHttp().getRequest().url
      }))
    );
  }
}
```

これらのインターセプターは、アプリケーション全体で一貫した動作を実現し、コードの重複を減らすのに役立ちます。また、ビジネスロジックと横断的関心事を明確に分離することができます。
