# パイプ

パイプは `@Injectable()` デコレータでアノテーションされ、`PipeTransform` インターフェースを実装したクラスです。

<figure>
  <img class="illustrative-image" src="/assets/Pipe_1.png" />
</figure>

パイプには2つの一般的なユースケースがあります:

- **変換**: 入力データを目的の形式に変換します(例: 文字列から整数へ)
- **バリデーション**: 入力データを評価し、有効な場合はそのまま通過させます。無効な場合は例外をスローします

どちらの場合も、パイプはコントローラーのルートハンドラーによって処理される引数に対して動作します。Nestはメソッドが呼び出される直前にパイプを介在させ、そのメソッドに渡される引数を受け取って操作します。変換やバリデーションの処理はその時点で行われ、その後(潜在的に)変換された引数でルートハンドラーが呼び出されます。

Nestには、すぐに使える組み込みパイプがいくつか用意されています。また、独自のカスタムパイプを作成することもできます。この章では、組み込みパイプを紹介し、それらをルートハンドラーにバインドする方法を説明します。その後、いくつかのカスタムパイプの実装を見て、ゼロからパイプを構築する方法を説明します。

> info **ヒント** パイプは例外ゾーン内で実行されます。つまり、パイプが例外をスローすると、例外レイヤー(グローバル例外フィルターと、現在のコンテキストに適用されている[例外フィルター](/exception-filters))によって処理されます。これを踏まえると、パイプで例外がスローされた場合、コントローラーメソッドは実行されないことが明確になります。これにより、外部ソースからアプリケーションに入ってくるデータを、システムの境界でバリデーションするためのベストプラクティス手法が得られます。

#### 組み込みパイプ

Nestには以下のパイプが最初から用意されています:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`
- `ParseDatePipe`

これらは `@nestjs/common` パッケージからエクスポートされています。

`ParseIntPipe` の使用例を簡単に見てみましょう。これは**変換**ユースケースの例で、メソッドハンドラーのパラメータがJavaScriptの整数に変換されることを保証します(変換に失敗した場合は例外をスローします)。この章の後半で、`ParseIntPipe` の簡単な独自実装を示します。以下の例のテクニックは、他の組み込み変換パイプ(`ParseBoolPipe`、`ParseFloatPipe`、`ParseEnumPipe`、`ParseArrayPipe`、`ParseDatePipe`、`ParseUUIDPipe` - この章では`Parse*`パイプと呼びます)にも適用できます。

#### パイプのバインド

パイプを使用するには、パイプクラスのインスタンスを適切なコンテキストにバインドする必要があります。`ParseIntPipe` の例では、パイプを特定のルートハンドラーメソッドに関連付け、メソッドが呼び出される前に実行されるようにします。これは以下のような構文で行います。これをメソッドパラメータレベルでのパイプのバインドと呼びます:

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

これにより、以下の2つの条件のいずれかが真であることが保証されます: `findOne()` メソッドで受け取るパラメータが数値である(期待通り `this.catsService.findOne()` を呼び出せる)か、ルートハンドラーが呼び出される前に例外がスローされるかのいずれかです。

例えば、ルートが以下のように呼び出されたとします:

```bash
GET localhost:3000/abc
```

Nestは以下のような例外をスローします:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

この例外により、`findOne()` メソッドの本文は実行されません。

上の例では、インスタンスではなくクラス(`ParseIntPipe`)を渡し、インスタンス化の責任をフレームワークに委ねて依存性注入を可能にしています。パイプやガードと同様に、代わりにインスタンスをインプレースで渡すこともできます。インプレースでインスタンスを渡すのは、組み込みパイプの動作をオプションを渡してカスタマイズしたい場合に便利です:

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

他の変換パイプ(すべての**Parse\***パイプ)のバインドも同様に機能します。これらのパイプはすべて、ルートパラメータ、クエリ文字列パラメータ、リクエストボディの値を検証する文脈で動作します。

例えば、クエリ文字列パラメータの場合:

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

以下は `ParseUUIDPipe` を使用して文字列パラメータをパースし、UUIDであるかを検証する例です:

```typescript
@@filename()
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
@@switch
@Get(':uuid')
@Bind(Param('uuid', new ParseUUIDPipe()))
async findOne(uuid) {
  return this.catsService.findOne(uuid);
}
```

> info **ヒント** `ParseUUIDPipe()` を使用すると、バージョン3、4、5のUUIDをパースできます。特定のバージョンのUUIDのみを要求する場合は、パイプのオプションでバージョンを指定できます。

ここまでで、様々な `Parse*` 系の組み込みパイプのバインドの例を見てきました。バリデーションパイプのバインドは少し異なります。これについては次のセクションで説明します。

> info **ヒント** バリデーションパイプの詳細な例については、[バリデーション手法](/techniques/validation)も参照してください。

#### カスタムパイプ

前述のように、独自のカスタムパイプを作成することができます。Nestは堅牢な組み込みの `ParseIntPipe` と `ValidationPipe` を提供していますが、それぞれの簡単なバージョンをゼロから構築してカスタムパイプの作成方法を見てみましょう。

まずは簡単な `ValidationPipe` から始めます。最初は、入力値を受け取ってすぐに同じ値を返す、恒等関数のように動作するようにします:

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class ValidationPipe {
  transform(value, metadata) {
    return value;
  }
}
```

> info **ヒント** `PipeTransform<T, R>` は、パイプが実装しなければならないジェネリックインターフェースです。ジェネリックインターフェースは、入力の `value` の型を示す `T` と、`transform()` メソッドの戻り値の型を示す `R` を使用します。

すべてのパイプは、`PipeTransform` インターフェースの契約を満たすために `transform()` メソッドを実装する必要があります。このメソッドには2つのパラメータがあります:

- `value`
- `metadata`

`value` パラメータは現在処理中のメソッド引数(ルートハンドラーメソッドによって受け取られる前)で、`metadata` は現在処理中のメソッド引数のメタデータです。メタデータオブジェクトには以下のプロパティがあります:

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

これらのプロパティは現在処理中の引数を記述します:

<table>
  <tr>
    <td>
      <code>type</code>
    </td>
    <td>引数が body <code>@Body()</code>、query <code>@Query()</code>、param <code>@Param()</code>、またはカスタムパラメータ(詳細は<a routerLink="/custom-decorators">こちら</a>)のいずれであるかを示します。</td>
  </tr>
  <tr>
    <td>
      <code>metatype</code>
    </td>
    <td>
      引数のメタタイプ(例: <code>String</code>)を提供します。注意: ルートハンドラーメソッドのシグネチャで型宣言を省略するか、バニラJavaScriptを使用する場合、値は <code>undefined</code> になります。
    </td>
  </tr>
  <tr>
    <td>
      <code>data</code>
    </td>
    <td>デコレータに渡された文字列(例: <code>@Body('string')</code>)です。デコレータの括弧を空にした場合は <code>undefined</code> になります。</td>
  </tr>
</table>

> warning **警告** TypeScriptのインターフェースはトランスパイル時に消えます。そのため、メソッドパラメータの型がクラスではなくインターフェースとして宣言されている場合、`metatype` の値は `Object` になります。

#### スキーマベースのバリデーション

バリデーションパイプをもう少し便利にしてみましょう。`CatsController` の `create()` メソッドをよく見てみると、サービスメソッドを実行しようとする前にポストボディオブジェクトが有効であることを確認したいと思うでしょう。

```typescript
@@filename()
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
async create(@Body() createCatDto) {
  this.catsService.create(createCatDto);
}
```

ボディパラメータ `createCatDto` に注目しましょう。その型は `CreateCatDto` です:

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

createメソッドに対する受信リクエストが有効なボディを含んでいることを確認したいと思います。そのため、`createCatDto` オブジェクトの3つのメンバーを検証する必要があります。これはルートハンドラーメソッド内で行うことができますが、そうすると**単一責任の原則**(SRP)に違反してしまいます。

別のアプローチとして、**バリデータクラス**を作成してそこにタスクを委譲することもできます。これには、各メソッドの先頭でこのバリデータを呼び出すことを忘れないようにしなければならないという欠点があります。

バリデーションミドルウェアを作成するのはどうでしょうか? これは機能する可能性はありますが、残念ながらアプリケーション全体のすべてのコンテキストで使用できる**汎用ミドルウェア**を作成することは不可能です。これは、ミドルウェアが呼び出されるハンドラーやそのパラメータなどの**実行コンテキスト**を認識できないためです。

もちろん、これはまさにパイプが設計されたユースケースです。では、バリデーションパイプを改良してみましょう。

<app-banner-courses></app-banner-courses>

#### オブジェクトスキーマバリデーション

クリーンで[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)な方法でオブジェクトバリデーションを行うためのアプローチがいくつかあります。一般的なアプローチの1つは**スキーマベース**のバリデーションです。このアプローチを試してみましょう。

[Zod](https://zod.dev/)ライブラリを使用すると、読みやすいAPIで簡単にスキーマを作成できます。Zodベースのスキーマを使用するバリデーションパイプを作成してみましょう。

必要なパッケージをインストールすることから始めます:

```bash
$ npm install --save zod
```

以下のコードサンプルでは、`constructor` 引数としてスキーマを受け取る簡単なクラスを作成します。次に、提供されたスキーマに対して受信引数を検証する。

```typescript
@@filename()
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
@@switch
import { BadRequestException } from '@nestjs/common';

export class ZodValidationPipe {
  constructor(private schema) {}

  transform(value, metadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

#### バリデーションパイプのバインド

先ほど、変換パイプ(`ParseIntPipe`や他の`Parse*`パイプなど)のバインド方法を見ました。

バリデーションパイプのバインドも非常に簡単です。

この場合、`ZodValidationPipe`を使用するために以下のことを行う必要があります：

1. `ZodValidationPipe`のインスタンスを作成する
2. パイプのクラスコンストラクタでコンテキスト固有のZodスキーマを渡す
3. パイプをメソッドにバインドする

Zodスキーマの例:

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

以下のように`@UsePipes()`デコレータを使用してバインドします：

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Bind(Body())
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **ヒント** `@UsePipes()`デコレータは`@nestjs/common`パッケージからインポートされます。

> warning **警告** `zod`ライブラリを使用するには、`tsconfig.json`ファイルで`strictNullChecks`設定を有効にする必要があります。

#### クラスバリデータ

> warning **警告** このセクションの手法はTypeScriptを必要とし、バニラJavaScriptで書かれたアプリケーションでは利用できません。

バリデーション技法の別の実装を見てみましょう。

Nestは[class-validator](https://github.com/typestack/class-validator)ライブラリとの相性が良いです。このパワフルなライブラリを使用すると、デコレータベースのバリデーションが可能になります。デコレータベースのバリデーションは、特にNestの**Pipe**機能と組み合わせると非常に強力です。なぜなら、処理されるプロパティの`metatype`にアクセスできるためです。始める前に、必要なパッケージをインストールする必要があります：

```bash
$ npm i --save class-validator class-transformer
```

これらをインストールしたら、`CreateCatDto`クラスにいくつかのデコレータを追加できます。ここでこの手法の大きな利点が見えます：`CreateCatDto`クラスは、Postボディオブジェクトの唯一の情報源として維持されます（別のバリデーションクラスを作成する必要はありません）。

```typescript
@@filename(create-cat.dto)
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> info **ヒント** class-validatorデコレータについての詳細は[こちら](https://github.com/typestack/class-validator#usage)をご覧ください。

これでこれらのアノテーションを使用する`ValidationPipe`クラスを作成できます。

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> info **ヒント** 繰り返しになりますが、汎用のバリデーションパイプを自分で構築する必要はありません。`ValidationPipe`はNestによってすぐに使える形で提供されています。この章で構築したサンプルよりも、組み込みの`ValidationPipe`の方が多くのオプションを提供しています。このサンプルは、カスタムビルドパイプの仕組みを説明するために基本的なものにとどめています。詳細な情報と多くの例は[こちら](/techniques/validation)で見つけることができます。


この単元「パイプ」の主要なユースケースを以下にまとめます：



1. **データ変換のユースケース**
   - 文字列から整数への変換（ParseIntPipe）
   - 文字列からブーリアンへの変換（ParseBoolPipe）
   - 文字列からUUIDへの変換と検証（ParseUUIDPipe）
   - 文字列から日付への変換（ParseDatePipe）
   - 配列データの解析（ParseArrayPipe）

2. **バリデーションのユースケース**
   - リクエストボディのバリデーション
   - DTOの検証（CreateCatDtoなどの入力データの検証）
   - スキーマベースのバリデーション（Zodを使用）
   - デコレータベースのバリデーション（class-validatorを使用）

3. **パイプのスコープ別使用例**
   - パラメータスコープ（特定のパラメータに対するバリデーション）
   - メソッドスコープ（特定のルートハンドラーに対するバリデーション）
   - グローバルスコープ（アプリケーション全体でのバリデーション）

4. **エラーハンドリングのユースケース**
   - バリデーション失敗時の例外スロー
   - カスタムエラーメッセージの提供
   - HTTPステータスコードのカスタマイズ

これらのユースケースは、アプリケーションの入力データの整合性を保証し、型安全性を確保するために重要な役割を果たします。パイプは特にAPIエンドポイントでデータを受け取る際の「門番」として機能し、不正なデータがアプリケーションの核となる部分に到達する前に検証と変換を行います。
