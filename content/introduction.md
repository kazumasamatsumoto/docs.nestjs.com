### はじめに

Nest（NestJS）は、効率的でスケーラブルな[Node.js](https://nodejs.org/)サーバーサイドアプリケーションを構築するためのフレームワークです。プログレッシブJavaScriptを使用し、[TypeScript](http://www.typescriptlang.org/)で構築され、完全にサポートしています（ただし、純粋なJavaScriptでのコーディングも可能です）。また、OOP（オブジェクト指向プログラミング）、FP（関数型プログラミング）、FRP（関数型リアクティブプログラミング）の要素を組み合わせています。

内部的には、Nestは[Express](https://expressjs.com/)（デフォルト）のような堅牢なHTTPサーバーフレームワークを使用し、オプションで[Fastify](https://github.com/fastify/fastify)を使用するように設定することもできます！

Nestは、これらの一般的なNode.jsフレームワーク（Express/Fastify）の上に抽象化レベルを提供しますが、同時に開発者に直接それらのAPIを公開します。これにより、開発者は基盤となるプラットフォームで利用可能な無数のサードパーティモジュールを自由に使用できます。

#### 哲学

近年、Node.jsのおかげで、JavaScriptはフロントエンドとバックエンドの両方のアプリケーションにおいて、ウェブの「共通言語」となりました。これにより、[Angular](https://angular.dev/)、[React](https://github.com/facebook/react)、[Vue](https://github.com/vuejs/vue)のような素晴らしいプロジェクトが生まれ、開発者の生産性を向上させ、高速で、テスト可能で、拡張可能なフロントエンドアプリケーションの作成を可能にしています。しかし、Node（およびサーバーサイドJavaScript）には優れたライブラリ、ヘルパー、ツールが多数存在するものの、それらはいずれも**アーキテクチャ**という主要な問題を効果的に解決していません。

Nestは、開発者やチームが高度にテスト可能で、スケーラブルで、疎結合で、保守が容易なアプリケーションを作成できるように、すぐに使えるアプリケーションアーキテクチャを提供します。このアーキテクチャは、Angularから大きな影響を受けています。

#### インストール

始めるには、[Nest CLI](/cli/overview)でプロジェクトをスキャフォールドするか、[スターター プロジェクトをクローン](#alternatives)することができます（どちらも同じ結果になります）。

Nest CLIでプロジェクトをスキャフォールドするには、以下のコマンドを実行します。これにより、新しいプロジェクトディレクトリが作成され、初期のNestコアファイルとサポートモジュールがディレクトリに配置され、プロジェクトの従来の基本構造が作成されます。**Nest CLI**で新しいプロジェクトを作成することは、初めてのユーザーにお勧めです。[はじめの一歩](first-steps)では、このアプローチを続けていきます。

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **ヒント** より厳密な機能セットを持つTypeScriptプロジェクトを作成するには、`nest new`コマンドに`--strict`フラグを渡してください。

#### 代替手段

あるいは、**Git**でTypeScriptスタータープロジェクトをインストールする場合：

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> info **ヒント** gitの履歴なしでリポジトリをクローンしたい場合は、[degit](https://github.com/Rich-Harris/degit)を使用できます。

ブラウザを開き、[`http://localhost:3000/`](http://localhost:3000/)に移動してください。

JavaScriptバージョンのスタータープロジェクトをインストールするには、上記のコマンドシーケンスで`javascript-starter.git`を使用してください。

また、コアパッケージとサポートパッケージをインストールして、新しいプロジェクトを最初から始めることもできます。ただし、プロジェクトのボイラープレートファイルは自分で設定する必要があります。最低限、`@nestjs/core`、`@nestjs/common`、`rxjs`、`reflect-metadata`の依存関係が必要です。最小限のNestJSアプリを一から作成する方法については、この短い記事をご覧ください：[5 steps to create a bare minimum NestJS app from scratch!](https://dev.to/micalevisk/5-steps-to-create-a-bare-minimum-nestjs-app-from-scratch-5c3b)。
