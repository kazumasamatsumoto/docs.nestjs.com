### モジュール

モジュールは`@Module()`デコレータでアノテーションされたクラスです。このデコレータは、**Nest**がアプリケーション構造を効率的に整理・管理するために使用するメタデータを提供します。

<figure><img class="illustrative-image" src="/assets/Modules_1.png" /></figure>

すべての Nest アプリケーションには、少なくとも 1 つのモジュール、つまり**ルートモジュール**があります。これは Nest が**アプリケーショングラフ**を構築するための起点となります。このグラフは、Nest がモジュールとプロバイダー間の関係や依存関係を解決するために使用する内部構造です。小規模なアプリケーションではルートモジュールのみで十分かもしれませんが、通常はそうではありません。モジュールは、コンポーネントを効果的に整理する方法として**強く推奨**されています。ほとんどのアプリケーションでは、密接に関連した**機能**のセットをカプセル化する複数のモジュールを持つことになるでしょう。

`@Module()`デコレータは、モジュールを記述するプロパティを持つ単一のオブジェクトを受け取ります：

|               |                                                                                                                                                                                     |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providers`   | Nest インジェクタによってインスタンス化され、少なくともこのモジュール内で共有される可能性のあるプロバイダー                                                                         |
| `controllers` | このモジュールで定義され、インスタンス化される必要のあるコントローラーのセット                                                                                                      |
| `imports`     | このモジュールで必要なプロバイダーをエクスポートするインポートされたモジュールのリスト                                                                                              |
| `exports`     | このモジュールによって提供され、他のモジュールでインポートして使用可能にすべきプロバイダーのサブセット。プロバイダー自体、またはそのトークン（`provide`値）のいずれかを使用できます |

モジュールは、デフォルトでプロバイダーを**カプセル化**します。つまり、現在のモジュールの一部であるか、他のインポートされたモジュールから明示的にエクスポートされたプロバイダーのみを注入できます。モジュールからエクスポートされたプロバイダーは、本質的にそのモジュールのパブリックインターフェースまたは API として機能します。

#### 機能モジュール

私たちの例では、`CatsController`と`CatsService`は密接に関連しており、同じアプリケーションドメインを提供しています。これらを機能モジュールとしてグループ化することは理にかなっています。機能モジュールは、特定の機能に関連するコードを整理し、明確な境界と優れた組織化を維持するのに役立ちます。これは、アプリケーションやチームが成長するにつれて特に重要になり、[SOLID](https://en.wikipedia.org/wiki/SOLID)原則にも沿っています。

次に、コントローラーとサービスをグループ化する方法を示すために`CatsModule`を作成します。

```typescript
@@filename(cats/cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> info **ヒント** CLI を使用してモジュールを作成するには、`$ nest g module cats`コマンドを実行するだけです。

上記では、`cats.module.ts`ファイルで`CatsModule`を定義し、このモジュールに関連するすべてのものを`cats`ディレクトリに移動しました。最後に必要なのは、このモジュールをルートモジュール（`app.module.ts`ファイルで定義された`AppModule`）にインポートすることです。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

現在のディレクトリ構造は以下のようになっています：

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
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### 共有モジュール

Nest では、モジュールはデフォルトで**シングルトン**であり、したがって複数のモジュール間で同じプロバイダーのインスタンスを簡単に共有することができます。

<figure><img class="illustrative-image" src="/assets/Shared_Module_1.png" /></figure>

すべてのモジュールは自動的に**共有モジュール**となります。一度作成されると、任意のモジュールで再利用できます。`CatsService`のインスタンスを他の複数のモジュール間で共有したいとしましょう。そのためには、まず以下のように`CatsService`プロバイダーをモジュールの`exports`配列に追加して**エクスポート**する必要があります：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

これで、`CatsModule`をインポートする任意のモジュールは`CatsService`にアクセスでき、それをインポートする他のすべてのモジュールと同じインスタンスを共有することになります。

もし`CatsService`を必要とする各モジュールで直接登録した場合、確かに動作はしますが、各モジュールが`CatsService`の独自のインスタンスを取得することになります。これにより、同じサービスの複数のインスタンスが作成されるためメモリ使用量が増加し、サービスが内部状態を維持している場合は状態の不整合などの予期しない動作を引き起こす可能性があります。

`CatsService`を`CatsModule`などのモジュール内にカプセル化してエクスポートすることで、`CatsModule`をインポートするすべてのモジュール間で同じ`CatsService`インスタンスが再利用されることを保証します。これはメモリ消費を削減するだけでなく、すべてのモジュールが同じインスタンスを共有するため、より予測可能な動作につながります。これにより、共有状態やリソースの管理が容易になります。これは、NestJS のようなフレームワークにおけるモジュール性と依存性注入の主要な利点の 1 つです。

<app-banner-devtools></app-banner-devtools>

#### モジュールの再エクスポート

上記で見たように、モジュールは内部のプロバイダーをエクスポートできます。さらに、インポートしたモジュールを再エクスポートすることもできます。以下の例では、`CommonModule`が`CoreModule`にインポートされ**かつ**エクスポートされており、このモジュールをインポートする他のモジュールでも使用できるようになっています。

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### 依存性の注入

モジュールクラスも同様にプロバイダーを**注入**することができます（例：設定目的など）：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
@@switch
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

ただし、[循環依存](/fundamentals/circular-dependency)の問題により、モジュールクラス自体をプロバイダーとして注入することはできません。

#### グローバルモジュール

同じモジュールセットをあらゆる場所でインポートする必要がある場合、それは面倒になる可能性があります。[Angular](https://angular.dev)とは異なり、Nest では`providers`はグローバルスコープに登録されません。一度定義されると、それらはどこでも利用可能です。しかし、Nest ではプロバイダーをモジュールスコープ内にカプセル化します。カプセル化しているモジュールを最初にインポートしない限り、モジュールのプロバイダーを他の場所で使用することはできません。

ヘルパー、データベース接続などのように、すぐに使用できるプロバイダーのセットを提供したい場合は、`@Global()`デコレータを使用してモジュールを**グローバル**にします。

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()`デコレータはモジュールをグローバルスコープにします。グローバルモジュールは**一度だけ**登録する必要があり、通常はルートモジュールまたはコアモジュールによって行われます。上記の例では、`CatsService`プロバイダーはどこでも利用可能となり、サービスを注入したいモジュールは`imports`配列に`CatsModule`をインポートする必要がなくなります。

> info **ヒント** すべてをグローバルにすることは、設計プラクティスとしては推奨されません。グローバルモジュールはボイラープレートを減らすのに役立ちますが、`imports`配列を使用してモジュールの API を他のモジュールで利用可能にする方が、より制御された明確な方法です。このアプローチは、より良い構造と保守性を提供し、モジュールの必要な部分のみが他と共有されることを保証し、アプリケーションの無関係な部分間の不要な結合を避けることができます。

#### 動的モジュール

Nest の動的モジュールを使用すると、実行時に設定可能なモジュールを作成できます。これは特に、プロバイダーが特定のオプションや設定に基づいて作成される必要がある、柔軟でカスタマイズ可能なモジュールを提供する必要がある場合に便利です。以下は**動的モジュール**の動作の概要です。

```typescript
@@filename()
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
@@switch
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> info **ヒント** `forRoot()`メソッドは、同期的または非同期的（つまり`Promise`を介して）に動的モジュールを返すことができます。

このモジュールは、デフォルトで`Connection`プロバイダーを定義します（`@Module()`デコレータのメタデータ内で）が、さらに - `forRoot()`メソッドに渡される`entities`と`options`オブジェクトに応じて - リポジトリなどのプロバイダーのコレクションを公開します。動的モジュールによって返されるプロパティは、`@Module()`デコレータで定義されたベースモジュールのメタデータを**上書き**するのではなく、**拡張**することに注意してください。これにより、静的に宣言された`Connection`プロバイダー**と**動的に生成されたリポジトリプロバイダーの両方がモジュールからエクスポートされます。

動的モジュールをグローバルスコープに登録したい場合は、`global`プロパティを`true`に設定します。

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> warning **警告** 上記で述べたように、すべてをグローバルにすることは**良い設計の決定ではありません**。

`DatabaseModule`は以下の方法でインポートおよび設定できます：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

動的モジュールを再エクスポートしたい場合は、exports 配列で`forRoot()`メソッドの呼び出しを省略できます：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[動的モジュール](/fundamentals/dynamic-modules)の章でこのトピックについてより詳しく説明しており、[動作例](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)も含まれています。

> info **ヒント** `ConfigurableModuleBuilder`を使用して高度にカスタマイズ可能な動的モジュールを構築する方法については、[この章](/fundamentals/dynamic-modules#configurable-module-builder)で学ぶことができます。
