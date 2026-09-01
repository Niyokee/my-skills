# 不正状態を表現できない型

複数項目の組み合わせや業務処理の段階を型として表現し、業務上存在しない状態を作れないようにする。

## 設計手順

1. 有効な状態と無効な状態を具体例で列挙する。
2. 型が表現できる状態の数と、業務上妥当な状態の数を数えて比較する。
   差がある場合、その差が不正な状態である。省略可能なフィールドが n 個あれば、
   組み合わせは 2 の n 乗になる。
3. 同時に必要な項目をAND条件としてまとめる。
4. 排他的または選択的な状態をOR条件として分ける。
5. 不変条件や利用可能な操作が異なる状態を、別の型にする。
6. 状態遷移を、入力状態から出力状態への関数として定義する。
7. 失敗し得る遷移を、結果型で表現する。

## ガイドライン

- AND条件には、必須フィールドを持つオブジェクト型を使う。
- OR条件には、判別可能な共用体を使う。
- 項目が存在しないこと自体に意味がある場合だけ、省略可能なフィールドを使う。
- 複数の真偽値で、排他的な状態を表現しない。
- 検証前と検証後のデータを同じ型にしない。
- 状態によって使用できないフィールドや操作を、共通型に押し込まない。
- 状態を識別する判別子に対して、網羅的な分岐を行う。
- 同じ役割のコレクションを並列に並べない。要素を追加しても既存の処理が
  壊れないため、対応漏れが検出されない。共通する概念を判別可能な共用体として
  抽出し、単一のコレクションにする。
- 選択肢の組み合わせが増えて共用体の場合分けが多くなったときは、
  場合を列挙する前に、まだ名前が付いていない概念がないか確認する。
- 「最低1件」の規則は、必須の1件と残りのコレクションに分けて表す。

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

## 並列するコレクションを統合する例

次の型は、連絡手段を追加しても既存の処理が壊れない。

```ts
type ContactInformation = Readonly<{
  emailAddresses: readonly EmailAddress[];
  postalAddresses: readonly PostalAddress[];
  phoneNumbers: readonly PhoneNumber[];
}>;
```

「メールの一覧と住所の一覧がある」ではなく、「連絡手段の一覧があり、
各連絡手段はメールまたは住所または電話である」と捉え直す。

```ts
type ContactMethod =
  | { readonly kind: "email"; readonly email: EmailAddress }
  | { readonly kind: "postal"; readonly address: PostalAddress }
  | { readonly kind: "phone"; readonly phone: PhoneNumber };

type ContactInformation = Readonly<{
  primary: ContactMethod;
  others: readonly ContactMethod[];
}>;
```

`ContactMethod`に選択肢を追加すると、すべての分岐が網羅性の検査で検出される。
また、`primary`を必須にすることで、連絡手段が一つもない状態を表現できなくする。

この過程で`ContactMethod`という、元の要件に現れていなかった概念が明らかになる。
型を厳密にしようとすると、モデルの誤りと不足している概念が現れる。

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

## スキーマ検証ライブラリでの共用体

zodでは`discriminatedUnion`を使う。判別子を指定することで、
検証の失敗理由が該当する場合分けに限定され、網羅性の検査も働く。

```ts
const ContactInfo = z.discriminatedUnion("kind", [
  z.object({ kind: z.literal("email"), email: EmailAddress }),
  z.object({ kind: z.literal("postal"), address: PostalAddress }),
]);
```

`z.union`を使うと、失敗時にすべての場合分けの誤りが列挙され、
どの状態を意図していたかが判別できない。判別子がある場合は
`discriminatedUnion`を使う。

Pythonでは、`Field`の判別子と型の合併で表す。

```python
from typing import Annotated, Literal, Union
from pydantic import BaseModel, Field

class EmailContact(BaseModel):
    kind: Literal["email"]
    email: EmailAddress

class PostalContact(BaseModel):
    kind: Literal["postal"]
    address: PostalAddress

ContactInfo = Annotated[
    Union[EmailContact, PostalContact],
    Field(discriminator="kind"),
]
```

分岐の網羅性は、型検査器の厳格な設定と`assert_never`で確認する。
既定の設定では検出されない。

## レビュー項目

- 任意項目の組み合わせによって不正状態を表現できないか。
- 複数の真偽値が同じ状態を重複して表していないか。
- 判別子と付随データの組み合わせが矛盾しないか。
- 検証前後や処理段階の異なるデータが同じ型になっていないか。
- 状態遷移を経ずに、後続状態を直接生成できないか。
- 新しい状態を追加した際に、未対応の分岐をコンパイラが検出できるか。
- 型の状態とデータベース上の状態が食い違う復元経路がないか。
- 同じ役割のコレクションが並列に並んでいないか。
- 表現できる状態の数と、業務上妥当な状態の数が一致しているか。
- スキーマ定義の共用体と、ドメインの共用体が同じ場合分けになっているか。

## 型だけでは保証できない範囲

型は、表現可能な状態と状態遷移の入口を制限できる。外部状態に依存する遷移条件、同時実行、永続化の原子性までは保証しない。それらはワークフロー、集約、トランザクションで扱う。
