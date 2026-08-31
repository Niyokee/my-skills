# 不正状態を表現できない型

複数項目の組み合わせや業務処理の段階を型として表現し、業務上存在しない状態を作れないようにする。

## 設計手順

1. 有効な状態と無効な状態を具体例で列挙する。
2. 同時に必要な項目をAND条件としてまとめる。
3. 排他的または選択的な状態をOR条件として分ける。
4. 不変条件や利用可能な操作が異なる状態を、別の型にする。
5. 状態遷移を、入力状態から出力状態への関数として定義する。
6. 失敗し得る遷移を、結果型で表現する。

## ガイドライン

- AND条件には、必須フィールドを持つオブジェクト型を使う。
- OR条件には、判別可能な共用体を使う。
- 項目が存在しないこと自体に意味がある場合だけ、省略可能なフィールドを使う。
- 複数の真偽値で、排他的な状態を表現しない。
- 検証前と検証後のデータを同じ型にしない。
- 状態によって使用できないフィールドや操作を、共通型に押し込まない。
- 状態を識別する判別子に対して、網羅的な分岐を行う。

## TypeScriptによるAND条件とOR条件の例

```ts
type ContactInfo =
  | {
      readonly kind: "email";
      readonly email: EmailAddress;
    }
  | {
      readonly kind: "postal";
      readonly address: PostalAddress;
    }
  | {
      readonly kind: "email-and-postal";
      readonly email: EmailAddress;
      readonly address: PostalAddress;
    };
```

`email?: EmailAddress`と`address?: PostalAddress`を並べると、両方が存在しない状態も表現できる。判別可能な共用体では、許可された三つの状態だけを表現する。

## TypeScriptによる状態遷移の例

```ts
type UnvalidatedOrder = Readonly<{
  shippingAddress: UnvalidatedAddress;
  lines: readonly UnvalidatedOrderLine[];
}>;

type ValidatedOrder = Readonly<{
  shippingAddress: ValidatedAddress;
  lines: readonly ValidatedOrderLine[];
}>;

type ValidateOrder = (
  order: UnvalidatedOrder,
) => Promise<Result<ValidatedOrder, readonly ValidationError[]>>;
```

検証後の処理には`ValidatedOrder`だけを渡す。検証済みかどうかを表す真偽値を`UnvalidatedOrder`に追加しない。

## 網羅性の確認例

```ts
function assertNever(value: never): never {
  throw new Error(`Unexpected state: ${String(value)}`);
}

function destination(info: ContactInfo): string {
  switch (info.kind) {
    case "email":
      return info.email.value;
    case "postal":
      return info.address.label;
    case "email-and-postal":
      return info.email.value;
    default:
      return assertNever(info);
  }
}
```

## レビュー項目

- 任意項目の組み合わせによって不正状態を表現できないか。
- 複数の真偽値が同じ状態を重複して表していないか。
- 判別子と付随データの組み合わせが矛盾しないか。
- 検証前後や処理段階の異なるデータが同じ型になっていないか。
- 状態遷移を経ずに、後続状態を直接生成できないか。
- 新しい状態を追加した際に、未対応の分岐をコンパイラが検出できるか。
- 型の状態とデータベース上の状態が食い違う復元経路がないか。

## 型だけでは保証できない範囲

型は、表現可能な状態と状態遷移の入口を制限できる。外部状態に依存する遷移条件、同時実行、永続化の原子性までは保証しない。それらはワークフロー、集約、トランザクションで扱う。
