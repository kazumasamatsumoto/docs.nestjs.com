### プロバイダー

プロバイダーは Nest の核となる概念です。サービス、リポジトリ、ファクトリー、ヘルパーなど、多くの基本的な Nest クラスはプロバイダーとして扱うことができます。プロバイダーの重要な考え方は、依存性として**注入**できることにあり、これによってオブジェクト同士が様々な関係を形成できるようになります。これらのオブジェクトの「配線」の責任は、主に Nest のランタイムシステムが担っています。

<figure><img class="illustrative-image" src="/assets/Components_1.png" /></figure>

前章では、シンプルな`CatsController`を作成しました。コントローラーは HTTP リクエストを処理し、より複雑なタスクを**プロバイダー**に委譲する必要があります。プロバイダーは、NestJS モジュールで`providers`として宣言されるプレーンな JavaScript クラスです。詳細については、「モジュール」の章を参照してください。

> info **ヒント** Nest はオブジェクト指向的な方法で依存関係を設計・整理することができるため、[SOLID の原則](https://en.wikipedia.org/wiki/SOLID)に従うことを強く推奨します。

#### サービス

まずは、シンプルな`CatsService`を作成してみましょう。このサービスはデータの保存と取得を処理し、`CatsController`で使用されます。アプリケーションのロジックを管理する役割を持つため、プロバイダーとして定義するのに理想的な候補となります。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> info **ヒント** CLI を使用してサービスを作成するには、`$ nest g service cats`コマンドを実行するだけです。

`CatsService`は 1 つのプロパティと 2 つのメソッドを持つ基本的なクラスです。ここでの重要な追加点は`@Injectable()`デコレータです。このデコレータはクラスにメタデータを付与し、`CatsService`が Nest の[IoC](https://en.wikipedia.org/wiki/Inversion_of_control)コンテナによって管理できるクラスであることを示します。

さらに、この例では`Cat`インターフェースを使用しています。これは以下のような形になるでしょう：

```typescript
@@filename(interfaces/cat.interface)
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

猫を取得するためのサービスクラスができたので、これを`CatsController`の中で使用してみましょう：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

The `CatsService` is **injected** through the class constructor. Notice the use of the `private` keyword. This shorthand allows us to both declare and initialize the `catsService` member in the same line, streamlining the process.

#### 依存性の注入

Nest は**依存性の注入**として知られる強力な設計パターンを中心に構築されています。この概念については、公式の[Angular ドキュメント](https://angular.dev/guide/di)にある優れた記事を読むことを強くお勧めします。

Nest では、TypeScript の機能のおかげで、依存関係の管理が簡単です。これは型に基づいて解決されるためです。以下の例では、Nest は`CatsService`のインスタンスを作成して返すことで`catsService`を解決します（シングルトンの場合、既に他の場所でリクエストされている場合は既存のインスタンスを返します）。この依存関係は、コントローラーのコンストラクタに注入されます（または指定されたプロパティに割り当てられます）：

```typescript
constructor(private catsService: CatsService) {}
```

#### スコープ

プロバイダーは通常、アプリケーションのライフサイクルに合わせたライフタイム（「スコープ」）を持ちます。アプリケーションが起動すると、各依存関係は解決される必要があり、つまりすべてのプロバイダーがインスタンス化されます。同様に、アプリケーションがシャットダウンすると、すべてのプロバイダーは破棄されます。ただし、プロバイダーを**リクエストスコープ**にすることも可能です。これは、そのライフタイムがアプリケーションのライフサイクルではなく、特定のリクエストに紐付けられることを意味します。これらのテクニックについては、[インジェクションスコープ](/fundamentals/injection-scopes)の章で詳しく学ぶことができます。

<app-banner-courses></app-banner-courses>

#### カスタムプロバイダー

Nest には、プロバイダー間の関係を管理する組み込みの制御の反転（「IoC」）コンテナが付属しています。この機能は依存性注入の基礎となるものですが、実際にはこれまで説明してきた以上に強力です。プロバイダーを定義する方法は複数あります：プレーンな値、クラス、そして非同期または同期のファクトリーを使用できます。プロバイダーの定義の詳細な例については、[依存性の注入](/fundamentals/dependency-injection)の章をご覧ください。

#### オプショナルプロバイダー

時には、必ずしも解決する必要のない依存関係がある場合があります。例えば、クラスが**設定オブジェクト**に依存しているが、それが提供されていない場合はデフォルト値を使用すべき、というような場合です。このような場合、依存関係はオプショナルと見なされ、設定プロバイダーが存在しないことはエラーとはなりません。

プロバイダーをオプショナルとしてマークするには、コンストラクタのシグネチャで`@Optional()`デコレータを使用します。

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

上記の例では、カスタムプロバイダーを使用しているため、カスタム**トークン**である`HTTP_OPTIONS`を含めています。これまでの例では、コンストラクタベースの注入を示してきました。これは、依存関係がコンストラクタ内のクラスを通じて示されるものです。カスタムプロバイダーとそれに関連するトークンの仕組みの詳細については、[カスタムプロバイダー](/fundamentals/custom-providers)の章をご覧ください。

#### プロパティベースの注入

これまで使用してきた手法は、コンストラクタベースの注入と呼ばれ、プロバイダーはコンストラクタメソッドを通じて注入されます。特定のケースでは、**プロパティベースの注入**が有用な場合があります。例えば、トップレベルのクラスが 1 つ以上のプロバイダーに依存している場合、サブクラスで`super()`を通じてそれらすべてを引き渡すのは面倒になる可能性があります。これを避けるため、プロパティレベルで直接`@Inject()`デコレータを使用できます。

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> warning **警告** クラスが他のクラスを継承していない場合は、通常**コンストラクタベース**の注入を使用する方が良いでしょう。コンストラクタは必要な依存関係を明確に指定し、`@Inject`でアノテーションされたクラスプロパティと比べて、より良い可視性とコードの理解のしやすさを提供します。

#### プロバイダーの登録

プロバイダー（`CatsService`）とコンシューマー（`CatsController`）を定義したので、注入を処理できるようにサービスを Nest に登録する必要があります。これは、モジュールファイル（`app.module.ts`）を編集し、`@Module()`デコレータの`providers`配列にサービスを追加することで行います。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

これで Nest は`CatsController`クラスの依存関係を解決できるようになります。

この時点で、ディレクトリ構造は以下のようになっているはずです：

<div class="file-tree">
<div class="item">src</div>
<div class="children">
<div class="item">cats</div>
<div class="children">
<div class="item">dto</div>
<div class="children">
<div class="item">create-cat.dto.ts</div>
</div>
<div class="item">interfaces</div>
<div class="children">
<div class="item">cat.interface.ts</div>
</div>
<div class="item">cats.controller.ts</div>
<div class="item">cats.service.ts</div>
</div>
<div class="item">app.module.ts</div>
<div class="item">main.ts</div>
</div>
</div>

#### 手動インスタンス化

これまで、Nest が依存関係の解決の詳細のほとんどを自動的に処理する方法について説明してきました。しかし、場合によっては組み込みの依存性注入システムの外に出て、プロバイダーを手動で取得またはインスタンス化する必要があるかもしれません。そのような技術について、以下で簡単に説明します。

- 既存のインスタンスを取得したり、プロバイダーを動的にインスタンス化したりするには、[モジュールリファレンス](https://docs.nestjs.com/fundamentals/module-ref)を使用できます。
- `bootstrap()`関数内でプロバイダーを取得するには（例：スタンドアロンアプリケーションの場合や、ブートストラップ中に設定サービスを使用する場合）、[スタンドアロンアプリケーション](https://docs.nestjs.com/standalone-applications)をご覧ください。
