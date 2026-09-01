# ユースケースワークフローの実装とレビュー

このガイドラインは、定義済みのユースケース要件を実装またはレビューするときに使用する。
ユースケース要件の記述方法は扱わない。

ここでは、ワークフローをユースケースの実装関数とする。

## 入力に関すること

ユースケースが業務上必要とする値を、1つの入力型として定義する。

```typescript
type PlaceOrderInput = Readonly<{
  orderId: string;
  customerId: string;
  lines: ReadonlyArray<{
    productCode: string;
    quantity: number;
  }>;
}>;
```

次の規則に従う。

- ユースケース要件に現れる業務入力だけを含める。
- 一体として扱う入力値は、1つのオブジェクト型にまとめる。
- 外部サービスやリポジトリなどの依存関係を、業務入力に含めない。
- 入力とドメイン内部の値で、不変条件、データ構造、または許可する操作が異なる場合は、別の型にする。
- 型を分ける理由がない場合は、処理段階ごとに機械的に型を増やさない。
- 実行者や実行時刻などは、ユースケース要件で必要な場合だけコマンド型に含める。
- `any`を使用しない。

```typescript
type Command<T> = Readonly<{
  data: T;
  requestedAt: Date;
  requestedBy: string;
}>;

type PlaceOrderCommand = Command<PlaceOrderInput>;
```

レビューでは、ユースケース要件との対応、必須項目と任意項目の区別、入力と内部状態を別の型にする理由を確認する。

## 出力に関すること

ユースケースが成功したときに保証する結果と、想定内の失敗を型として定義する。

```typescript
type PlaceOrderEvent =
  | Readonly<{ type: "OrderPlaced"; orderId: string }>
  | Readonly<{
      type: "BillableOrderPlaced";
      orderId: string;
      amount: number;
    }>;

type PreparationError = Readonly<{ reason: string }>;
type PricingError = Readonly<{ reason: string }>;

type PlaceOrderError =
  | Readonly<{ type: "PreparationFailed"; error: PreparationError }>
  | Readonly<{ type: "PricingFailed"; error: PricingError }>
  | Readonly<{ type: "DependencyUnavailable"; dependency: string }>;

type PlaceOrderOutput = ReadonlyArray<PlaceOrderEvent>;
type PlaceOrderResult = Result<PlaceOrderOutput, PlaceOrderError>;
```

次の規則に従う。

### 失敗の分類

すべての失敗を`Result`型に含めない。失敗を三つに分類し、扱いを変える。

- **業務上の失敗**: 業務プロセスの一部として想定される失敗。与信の否認、
  在庫の不足、無効な製品コードなど。業務側に対処手順が存在する。
  判別可能な共用体として列挙し、ユースケースのエラー型に含める。
- **回復不能な失敗**: メモリ不足、ゼロ除算、null参照など。
  型に表さず、例外として最上位で扱う。
- **基盤の失敗**: 通信のタイムアウト、認証の失敗など。想定はされるが
  業務プロセスの一部ではない。業務上の失敗と同じ型に混ぜない。
  呼び出し側に伝える必要がある場合だけ、別の選択肢として含める。

分類に迷う場合は、「この失敗が起きたとき、業務としてどう対処するか」を確認する。
対処手順が定まっているなら業務上の失敗である。定まっていないなら、
業務上の失敗として型に含める前に、業務側へ確認する。

エラーの内容を文字列だけで表さない。呼び出し側が種類で分岐できず、
種類を追加しても既存の分岐が壊れない。

- 成功結果を専用の出力型として定義する。
- 複数の値を同時に返す場合は、1つのオブジェクト型にまとめる。
- 排他的な結果を判別可能なユニオン型で表す。
- 後続処理へ通知する事実を、過去形のイベントとして表す。
- 想定内の失敗を例外ではなく`Result`型で表す。
- `Result`は汎用的に定義し、エラーはユースケース固有の型にする。

複数のエラーを返す場合は、すべてのエラーを蓄積するのか、最初のエラーで停止するのかを明示する。

レビューでは、成功時の保証、排他的な結果、想定内の失敗が、それぞれ型から判断できるかを確認する。

## 処理内容に関すること

ユースケースを、型で接続できる小さなステップへ分ける。

ステップには、ドメインの振る舞い、ドメインサービス、外部依存との接続処理が含まれる。各ステップを組み合わせる処理がユースケースである。

処理の前後で不変条件、データ構造、または許可する操作が変わる場合は、中間状態を別の型として表す。単に処理を通過したことだけを理由に、型を分けない。

```typescript
type OrderReadyForPricing = Readonly<{
  /* 価格計算に必要な不変条件を満たすドメイン値 */
}>;
type PricedOrder = Readonly<{ /* 価格計算済みのドメイン値 */ }>;

type PrepareOrder = (
  input: PlaceOrderInput,
) => ResultAsync<OrderReadyForPricing, PreparationError>;

type PriceOrder = (
  input: OrderReadyForPricing,
) => ResultAsync<PricedOrder, PricingError>;

type CreateEvents = (input: PricedOrder) => PlaceOrderOutput;
```

外部依存は目的を表す関数型として定義し、組み立て時に注入する。

```typescript
type PlaceOrderDependencies = Readonly<{
  productExists: (productCode: ProductCode) => Promise<boolean>;
  getProductPrice: (productCode: ProductCode) => Promise<Money>;
}>;

type PlaceOrder = (
  command: PlaceOrderCommand,
) => ResultAsync<PlaceOrderOutput, PlaceOrderError>;

type MakePlaceOrder = (
  dependencies: PlaceOrderDependencies,
) => PlaceOrder;
```

ワークフロー本体は、成功値を次のステップへ渡す。失敗した場合は後続ステップを実行せず、ステップ固有のエラーをユースケース固有のエラーへ変換する。

`Result`ライブラリが提供する成功値の変換、エラーの変換、次の処理への接続を使用する。以下の`map`、`mapErr`、`andThen`は、特定のライブラリではなく、それぞれに相当する操作を表す。

```typescript
const makePlaceOrder: MakePlaceOrder = (dependencies) => {
  const prepareOrder = makePrepareOrder(dependencies.productExists);
  const priceOrder = makePriceOrder(dependencies.getProductPrice);

  return (command) =>
    prepareOrder(command.data)
      .mapErr(
        (error): PlaceOrderError => ({
          type: "PreparationFailed",
          error,
        }),
      )
      .andThen((order) =>
        priceOrder(order).mapErr(
          (error): PlaceOrderError => ({
            type: "PricingFailed",
            error,
          }),
        ),
      )
      .map(createEvents);
};
```

次の規則に従う。

- 各ステップの出力型を、次のステップの入力型に一致させる。
- 各ステップの失敗と非同期性を、関数シグネチャに含める。
- 外部との入出力、時刻、乱数などを依存関数として注入する。
- 依存関係を注入する組み立て関数と、入力を処理するユースケース関数を分離する。
- 同じ成功・失敗の分岐を手書きせず、`Result`ライブラリの合成操作を使用する。
- 取りうるすべての入力に対して出力が決まる関数として設計する。
  例外を投げない、値がないことを黙って表さない、隠れた入出力を行わない、
  隠れた依存を持たない、の四つを満たす。
- 失敗の可能性を型で表す代わりに、入力の型を狭めて失敗自体をなくせないか確認する。
  ゼロ除算を結果型で扱うより、ゼロ以外であることを保証した型を受け取るほうが、
  呼び出し側の分岐が減る。

レビューでは、各ステップの責務、中間状態の型、外部依存の明示、後続処理の停止、ユースケース固有のエラーへの変換を確認する。

## 型だけでは保証できない範囲

型は、各段階の入出力、想定内の失敗、非同期性、依存関係を表せる。
実行時の競合、再試行による重複適用、メッセージの消失、
複数集約にまたがる保存の原子性は保証しない。
それらは`design-guidelines`で扱う。
