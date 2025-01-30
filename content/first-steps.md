### はじめの一歩

この一連の記事では、Nestの**コアとなる基礎**について学びます。Nestアプリケーションの基本的な構成要素に慣れるために、入門レベルで広範な機能をカバーする基本的なCRUDアプリケーションを構築していきます。

#### プログラミング言語

私たちは[TypeScript](https://www.typescriptlang.org/)を愛用していますが、何よりも[Node.js](https://nodejs.org/en/)を愛しています。そのため、NestはTypeScriptと純粋なJavaScriptの両方に対応しています。Nestは最新の言語機能を活用しているため、バニラJavaScriptで使用する場合は[Babel](https://babeljs.io/)コンパイラが必要です。

提供する例では主にTypeScriptを使用しますが、各コードスニペットの右上隅にある言語切り替えボタンをクリックすることで、いつでもバニラJavaScriptの構文に**切り替える**ことができます。

#### 前提条件

お使いのオペレーティングシステムに[Node.js](https://nodejs.org)（バージョン20以上）がインストールされていることを確認してください。

#### セットアップ

[Nest CLI](/cli/overview)を使用すると、新しいプロジェクトのセットアップは非常に簡単です。[npm](https://www.npmjs.com/)をインストールした状態で、OSのターミナルで以下のコマンドを実行することで新しいNestプロジェクトを作成できます：

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **ヒント** TypeScriptの[より厳格な](https://www.typescriptlang.org/tsconfig#strict)機能セットで新しいプロジェクトを作成するには、`nest new`コマンドに`--strict`フラグを付けてください。

`project-name`ディレクトリが作成され、node modulesといくつかの他のボイラープレートファイルがインストールされ、`src/`ディレクトリが作成され、いくつかのコアファイルが配置されます。

```
src/
├── app.controller.spec.ts
├── app.controller.ts
├── app.module.ts
├── app.service.ts
└── main.ts
```

これらのコアファイルの簡単な概要です：

|                          |                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `app.controller.ts`      | 単一のルートを持つ基本的なコントローラー。                                                                           |
| `app.controller.spec.ts` | コントローラーのユニットテスト。                                                                                     |
| `app.module.ts`          | アプリケーションのルートモジュール。                                                                                 |
| `app.service.ts`         | 単一のメソッドを持つ基本的なサービス。                                                                               |
| `main.ts`                | Nestアプリケーションインスタンスを作成するコア機能`NestFactory`を使用するアプリケーションのエントリーファイル。      |

`main.ts`には、アプリケーションを**ブートストラップ**する非同期関数が含まれています：

```typescript
@@filename(main)

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Nestアプリケーションインスタンスを作成するために、コアの`NestFactory`クラスを使用します。`NestFactory`は、アプリケーションインスタンスを作成できるいくつかの静的メソッドを公開しています。`create()`メソッドは、`INestApplication`インターフェースを実装するアプリケーションオブジェクトを返します。このオブジェクトは、後の章で説明するメソッド群を提供します。上記の`main.ts`の例では、単にHTTPリスナーを起動して、アプリケーションがインバウンドのHTTPリクエストを待ち受けられるようにしています。

Nest CLIでスキャフォールドされたプロジェクトは、各モジュールを専用のディレクトリに保持するという規約に従うことを推奨する初期プロジェクト構造を作成することに注意してください。

> info **ヒント** デフォルトでは、アプリケーションの作成中にエラーが発生した場合、アプリはコード`1`で終了します。代わりにエラーをスローさせたい場合は、`abortOnError`オプションを無効にしてください（例：`NestFactory.create(AppModule, {{ '{' }} abortOnError: false {{ '}' }})`）。

<app-banner-courses></app-banner-courses>

#### プラットフォーム

Nestは、プラットフォームに依存しないフレームワークを目指しています。プラットフォームの独立性により、開発者が異なる種類のアプリケーション間で活用できる再利用可能な論理部分を作成することが可能になります。技術的には、Nestはアダプターが作成されれば任意のNode HTTPフレームワークで動作することができます。[express](https://expressjs.com/)と[fastify](https://www.fastify.io)という2つのHTTPプラットフォームが標準でサポートされています。ニーズに最も適したものを選択できます。

|                    |                                                                                                                                                                                                                                                                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `platform-express` | [Express](https://expressjs.com/)は、node用のよく知られたミニマリストWebフレームワークです。これは実戦で検証された本番環境対応のライブラリで、コミュニティによって実装された多くのリソースがあります。`@nestjs/platform-express`パッケージがデフォルトで使用されます。多くのユーザーにとってExpressは十分であり、有効にするための特別な操作は必要ありません。 |
| `platform-fastify` | [Fastify](https://www.fastify.io/)は、最大の効率性とスピードを提供することに重点を置いた、高性能で低オーバーヘッドのフレームワークです。使用方法については[こちら](/techniques/performance)をご覧ください。                                                                                                                                  |

使用されるプラットフォームに関係なく、それぞれ独自のアプリケーションインターフェースを公開します。これらはそれぞれ`NestExpressApplication`と`NestFastifyApplication`として見ることができます。

以下の例のように`NestFactory.create()`メソッドに型を渡すと、`app`オブジェクトはそのプラットフォーム固有のメソッドを使用できるようになります。ただし、基盤となるプラットフォームAPIにアクセスしたい場合を**除いて**、型を指定する必要は**ありません**。

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

#### アプリケーションの実行

インストールプロセスが完了したら、OSのコマンドプロンプトで以下のコマンドを実行して、インバウンドのHTTPリクエストを待ち受けるアプリケーションを起動できます：

```bash
$ npm run start
```

> info **ヒント** 開発プロセスを高速化するために（ビルドが20倍速く）、`start`スクリプトに`-b swc`フラグを付けて[SWCビルダー](/recipes/swc)を使用できます：`npm run start -- -b swc`。

このコマンドは、`src/main.ts`ファイルで定義されたポートでHTTPサーバーを待ち受けるアプリケーションを起動します。アプリケーションが起動したら、ブラウザを開いて`http://localhost:3000/`にアクセスしてください。`Hello World!`メッセージが表示されるはずです。

ファイルの変更を監視するには、以下のコマンドを実行してアプリケーションを起動できます：

```bash
$ npm run start:dev
```

このコマンドはファイルを監視し、自動的にサーバーを再コンパイルして再起動します。

#### リンティングとフォーマッティング

[CLI](/cli/overview)は、大規模な開発ワークフローを確実にスキャフォールドするための最善の努力を提供します。そのため、生成されたNestプロジェクトには、コード**リンター**と**フォーマッター**（それぞれ[eslint](https://eslint.org/)と[prettier](https://prettier.io/)）が事前にインストールされています。

> info **ヒント** フォーマッターとリンターの役割の違いがわからない場合は、[こちら](https://prettier.io/docs/en/comparison.html)で違いを学んでください。

最大限の安定性と拡張性を確保するために、ベースとなる[`eslint`](https://www.npmjs.com/package/eslint)と[`prettier`](https://www.npmjs.com/package/prettier)のcliパッケージを使用しています。このセットアップにより、公式の拡張機能との優れたIDE統合が設計上可能になります。

IDEが関係ないヘッドレス環境（継続的インテグレーション、Gitフック等）向けに、Nestプロジェクトには使用準備の整った`npm`スクリプトが付属しています。

```bash
# eslintでリントと自動修正を行う
$ npm run lint

# prettierでフォーマットする
$ npm run format
```
