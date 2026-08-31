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

レビューでは、各ステップの責務、中間状態の型、外部依存の明示、後続処理の停止、ユースケース固有のエラーへの変換を確認する。
