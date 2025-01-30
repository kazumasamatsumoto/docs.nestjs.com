

### カスタムプロバイダー

これまでの章で、**依存性注入(DI)** の様々な側面とその使用方法について触れてきました。その一例として、クラスにインスタンス（多くの場合はサービスプロバイダー）を注入するための[コンストラクタベース](https://docs.nestjs.com/providers#dependency-injection)の依存性注入があります。依存性注入がNestのコアに根本的な方法で組み込まれているということは、驚くことではないでしょう。これまでは主要なパターンを1つだけ見てきました。アプリケーションが複雑になるにつれ、DIシステムの全機能を活用する必要が出てくるかもしれません。そこで、それらについて詳しく見ていきましょう。

#### DI の基礎

依存性注入は[制御の反転（IoC）](https://en.wikipedia.org/wiki/Inversion_of_control)の手法で、依存関係のインスタンス化を命令的にコードで行うのではなく、IoCコンテナ（この場合はNestJSのランタイムシステム）に委譲します。[プロバイダーの章](https://docs.nestjs.com/providers)の例を見てみましょう。

まず、プロバイダーを定義します。`@Injectable()`デコレータは`CatsService`クラスをプロバイダーとしてマークします。

```typescript
// cats.service.ts
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

次に、Nestにプロバイダーをコントローラークラスに注入するよう要求します：

```typescript
// cats.controller.ts
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

最後に、プロバイダーをNest IoCコンテナに登録します：

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

これを機能させるために、裏で何が起きているのでしょうか？このプロセスには3つの重要なステップがあります：

1. `cats.service.ts`で、`@Injectable()`デコレータが`CatsService`クラスをNest IoCコンテナで管理できるクラスとして宣言します。

2. `cats.controller.ts`で、`CatsController`がコンストラクタ注入を使用して`CatsService`トークンへの依存関係を宣言します：

```typescript
constructor(private catsService: CatsService)
```

3. `app.module.ts`で、トークン`CatsService`を`cats.service.ts`ファイルの`CatsService`クラスと関連付けます。この関連付け（登録とも呼ばれる）がどのように行われるかは<a href="/fundamentals/custom-providers#standard-providers">以下</a>で説明します。

Nest IoCコンテナが`CatsController`をインスタンス化する際、まず依存関係を探します。`CatsService`依存関係を見つけると、`CatsService`トークンの検索を実行し、登録ステップ（上記の#3）に従って`CatsService`クラスを返します。`SINGLETON`スコープ（デフォルトの動作）を想定すると、Nestは`CatsService`のインスタンスを作成してキャッシュし、それを返すか、すでにキャッシュされている場合は既存のインスタンスを返します。

*この説明は要点を説明するために少し簡略化されています。重要な点として、依存関係のコード分析のプロセスは非常に高度で、アプリケーションの起動時に行われます。重要な機能の1つは、依存関係の分析（または「依存関係グラフの作成」）が**推移的**であるということです。上記の例では、`CatsService`自体に依存関係があった場合、それらも解決されます。依存関係グラフは、依存関係が正しい順序で解決されることを保証します - 基本的に「ボトムアップ」です。このメカニズムにより、開発者はこのような複雑な依存関係グラフを管理する必要がなくなります。

#### 標準プロバイダー

`@Module()`デコレータをより詳しく見てみましょう。`app.module`では、以下のように宣言しています：

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers`プロパティは`providers`の配列を受け取ります。これまで、クラス名のリストでこれらのプロバイダーを提供してきました。実際、`providers: [CatsService]`の構文は、より完全な構文の省略形です：

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

この明示的な構造を見ると、登録プロセスを理解できます。ここでは、トークン`CatsService`をクラス`CatsService`と明確に関連付けています。省略形の表記は、トークンが同じ名前のクラスのインスタンスを要求するという最も一般的なユースケースを簡略化するための便利な方法に過ぎません。

#### カスタムプロバイダー

_標準プロバイダー_が提供する機能以上のものが必要な場合はどうすればよいでしょうか？以下にいくつかの例を示します：

- Nestにクラスをインスタンス化させる（またはキャッシュされたインスタンスを返させる）代わりに、カスタムインスタンスを作成したい
- 既存のクラスを2番目の依存関係で再利用したい
- テスト用にクラスをモックバージョンで上書きしたい

Nestではこれらのケースを処理するためのカスタムプロバイダーを定義できます。カスタムプロバイダーを定義するためのいくつかの方法を提供しています。それらを見ていきましょう。

> info **ヒント** 依存関係の解決に問題がある場合は、`NEST_DEBUG`環境変数を設定することで、起動時に追加の依存関係解決ログを取得できます。

#### 値プロバイダー：`useValue`

`useValue`構文は、定数値を注入したり、外部ライブラリをNestコンテナに入れたり、実際の実装をモックオブジェクトで置き換えたりする場合に便利です。例えば、テスト目的でNestにモックの`CatsService`を強制的に使用させたい場合を考えてみましょう：

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* モック実装
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

この例では、`CatsService`トークンは`mockCatsService`モックオブジェクトに解決されます。`useValue`には値が必要です - この場合、置き換える`CatsService`クラスと同じインターフェースを持つリテラルオブジェクトです。TypeScriptの[構造的型付け](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)のおかげで、リテラルオブジェクトや`new`でインスタンス化されたクラスインスタンスを含む、互換性のあるインターフェースを持つオブジェクトを使用できます。

#### クラスベースではないプロバイダートークン

これまで、プロバイダートークン（`providers`配列にリストされているプロバイダーの`provide`プロパティの値）としてクラス名を使用してきました。これは[コンストラクタベースの注入](https://docs.nestjs.com/providers#dependency-injection)で使用される標準パターンと一致します。（トークンの概念が完全に明確でない場合は、<a href="/fundamentals/custom-providers#di-fundamentals">DI の基礎</a>を参照してください）。場合によっては、DIトークンとして文字列やシンボルを使用する柔軟性が必要になることがあります。例えば：

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

この例では、文字列値のトークン（`'CONNECTION'`）を、外部ファイルからインポートした既存の`connection`オブジェクトと関連付けています。

> warning **注意** トークン値として文字列を使用することに加えて、JavaScript[シンボル](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)やTypeScript[列挙型](https://www.typescriptlang.org/docs/handbook/enums.html)も使用できます。

これまで、標準の[コンストラクタベースの注入](https://docs.nestjs.com/providers#dependency-injection)パターンを使用してプロバイダーを注入する方法を見てきました。このパターンでは、依存関係をクラス名で宣言する必要があります。`'CONNECTION'`カスタムプロバイダーは文字列値のトークンを使用します。このようなプロバイダーを注入する方法を見てみましょう。そのために、`@Inject()`デコレータを使用します。このデコレータは単一の引数 - トークン - を取ります。

```typescript
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

> info **ヒント** `@Inject()`デコレータは`@nestjs/common`パッケージからインポートされます。

上記の例では説明のために直接文字列`'CONNECTION'`を使用していますが、クリーンなコード組織化のために、トークンは`constants.ts`のような別のファイルで定義するのがベストプラクティスです。それらを、独自のファイルで定義され、必要な場所でインポートされるシンボルや列挙型と同様に扱います。

#### エイリアスプロバイダー：`useExisting`

`useExisting`構文を使用すると、既存のプロバイダーのエイリアスを作成できます。これにより、同じプロバイダーにアクセスする2つの方法が作成されます。以下の例では、（文字列ベースの）トークン`'AliasedLoggerService'`は（クラスベースの）トークン`LoggerService`のエイリアスです。`'AliasedLoggerService'`と`LoggerService`の2つの異なる依存関係があるとします。両方の依存関係が`SINGLETON`スコープで指定されている場合、両方とも同じインスタンスに解決されます。

```typescript
@Injectable()
class LoggerService {
  /* 実装の詳細 */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

#### サービスベースではないプロバイダー

プロバイダーはしばしばサービスを提供しますが、その用途に限定されるものではありません。プロバイダーは**どんな**値でも提供できます。例えば、プロバイダーは現在の環境に基づいて設定オブジェクトの配列を提供することができます：

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

#### カスタムプロバイダーのエクスポート

他のプロバイダーと同様に、カスタムプロバイダーは宣言したモジュールにスコープされています。他のモジュールから見えるようにするには、エクスポートする必要があります。カスタムプロバイダーをエクスポートするには、そのトークンまたはプロバイダーオブジェクト全体を使用できます。

以下の例は、トークンを使用したエクスポートを示しています：

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

あるいは、プロバイダーオブジェクト全体をエクスポートすることもできます：

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```

これで、NestJSのカスタムプロバイダーに関する文書の翻訳が完了しました。この文書では、様々な種類のカスタムプロバイダーとその使用方法について詳しく説明しています。

NestJSのカスタムプロバイダーの主なユースケースをまとめさせていただきます。

### カスタムプロバイダーの主なユースケース

1. **値の直接注入** (`useValue`)
   - テスト時のモックオブジェクトの注入
   - 設定値や定数の注入
   - 外部ライブラリのインスタンスの注入
   ```typescript
   {
     provide: CatsService,
     useValue: mockCatsService
   }
   ```

2. **動的なクラス選択** (`useClass`)
   - 環境に応じた異なる実装の切り替え（開発/本番環境）
   - 機能フラグに基づく実装の切り替え
   ```typescript
   {
     provide: ConfigService,
     useClass: process.env.NODE_ENV === 'development' 
       ? DevConfigService 
       : ProdConfigService
   }
   ```

3. **動的な値の生成** (`useFactory`)
   - 実行時の条件に基づく値の生成
   - 複数の依存関係を組み合わせた値の生成
   - 非同期的な初期化が必要な場合
   ```typescript
   {
     provide: 'DATABASE_CONNECTION',
     useFactory: async (config: ConfigService) => {
       return await createConnection(config.dbConfig);
     },
     inject: [ConfigService]
   }
   ```

4. **エイリアスの作成** (`useExisting`)
   - 既存のサービスに別名を付ける
   - 後方互換性の維持
   - インターフェースの抽象化
   ```typescript
   {
     provide: 'LoggerAliasToken',
     useExisting: LoggerService
   }
   ```

5. **非サービス値の提供**
   - 環境設定オブジェクト
   - 定数や設定値の集合
   - アプリケーション全体で共有される値
   ```typescript
   {
     provide: 'APP_CONFIG',
     useFactory: () => ({
       apiUrl: process.env.API_URL,
       timeout: 3000
     })
   }
   ```

これらのユースケースを適切に組み合わせることで、より柔軟でメンテナンス性の高いアプリケーションを構築することができます。特に、テスト容易性の向上や、環境に応じた動的な振る舞いの実現に役立ちます。
