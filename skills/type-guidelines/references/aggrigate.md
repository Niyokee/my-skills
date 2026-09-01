# 集約に関する型設計

このガイドラインは、集約に関して型で表現できる規則だけを扱う。トランザクション、永続化、メッセージ配送、結果整合性は扱わない。

## 1. 同時に整合させるデータを一つの集約型にまとめる

関連するデータの間で成立させる業務規則を明記する。その規則が破られた状態を具体例として示す。

例えば、注文の請求金額は全注文明細の合計と一致させる。明細の合計が3,000円で、請求金額が2,500円なら、この業務規則は破られている。

このように、一つの操作によって整合させる必要があるデータは、一つの集約型に含める。注文明細と請求金額を別々の型として外部へ公開しない。

```ts
type NonEmptyLines = readonly [OrderLine, ...OrderLine[]];

type OrderState = Readonly<{
  id: OrderId;
  lines: NonEmptyLines;
  amountToBill: Money;
}>;
```

## 2. 整合性を保つ関数だけを公開する

集約型を構成する値を、外部から個別に変更させない。集約全体を受け取り、業務規則を保った新しい集約全体を返す関数だけを公開する。

集約型のコンストラクタは非公開にする。生成関数と更新関数は、関連する値を同時に計算する。

```ts
type Result<T, E> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

export class Order {
  private constructor(
    readonly id: OrderId,
    readonly lines: NonEmptyLines,
    readonly amountToBill: Money,
  ) {}

  static create(id: OrderId, lines: NonEmptyLines): Order {
    return new Order(id, lines, sumLinePrices(lines));
  }

  changeLinePrice(
    lineId: OrderLineId,
    newPrice: Money,
  ): Result<Order, "not-found"> {
    const changed = replaceLinePrice(this.lines, lineId, newPrice);

    if (!changed.ok) {
      return changed;
    }

    return {
      ok: true,
      value: new Order(
        this.id,
        changed.value,
        sumLinePrices(changed.value),
      ),
    };
  }
}
```

`amountToBill`だけを変更する関数や、`OrderLine`だけを置き換える関数を、集約の外部へ公開しない。

## 3. 独立した業務概念を新しいエンティティ型として表現する

複数の既存エンティティを変更する処理が、独自の識別子、属性、業務規則、またはライフサイクルを持つか確認する。該当する場合は、その処理を既存型の操作だけで表現せず、新しいエンティティ型として表現する。

例えば、口座間の送金が送金識別子を持つなら、二つの口座を変更するだけの処理ではなく、`MoneyTransfer`というエンティティとして表現する。

```ts
type MoneyTransfer = Readonly<{
  id: MoneyTransferId;
  fromAccount: AccountId;
  toAccount: AccountId;
  amount: PositiveMoney;
}>;
```

既存の型を再利用することより、業務上存在する概念を型として明示することを優先する。

## 4. 共通する制約を制約付き型として共有する

複数の集約が同じ値の制約を必要とする場合は、同じ検証処理を各集約へ重複させず、制約を表す型を共有する。

例えば、複数の集約で金額が0以上でなければならない場合は、`number`と個別の検証処理ではなく、`NonNegativeMoney`を使う。

```ts
export class NonNegativeMoney {
  private constructor(readonly value: number) {}

  static create(
    value: number,
  ): Result<NonNegativeMoney, "not-finite" | "negative"> {
    if (!Number.isFinite(value)) {
      return { ok: false, error: "not-finite" };
    }

    if (value < 0) {
      return { ok: false, error: "negative" };
    }

    return { ok: true, value: new NonNegativeMoney(value) };
  }
}
```

共通する業務規則だけを共有する。名前が似ていても、不変条件が異なる値を同じ型にまとめない。

## レビュー項目

- 同時に整合させるデータが、別々の型として個別に更新可能になっていないか。
- 集約の内部値を、整合性を保つ関数を通さず変更できないか。
- 複数の既存エンティティにまたがる処理が、独立した業務概念を示していないか。
- 同じ値の制約が複数の集約に重複していないか。
- 不変条件の異なる値を、再利用のために同じ制約付き型へまとめていないか。
