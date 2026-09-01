# 単一値の完全性

完全性とは、値がその値に適用される業務規則を常に満たしていることを指す。検証済みの値だけをドメイン内部で扱えるようにする。

## 設計手順

1. 値の意味、単位、許容範囲、形式、正規化規則を確認する。
2. 同じプリミティブ型でも、意味または不変条件が異なる値を別の型にする。
3. 通常のコンストラクタを公開せず、スマートコンストラクタだけを公開する。
4. スマートコンストラクタで全不変条件を検証し、失敗を戻り値の型で表現する。
5. 作成後の値を不変にする。
6. 外部入力、データベースからの復元、メッセージ受信でも同じ生成経路を使う。

## ガイドライン

- 重要な業務値を、裸の文字列や数値のまま受け渡さない。
- コンストラクタの非公開化とスマートコンストラクタを組み合わせる。
- 検証失敗が想定内なら、例外ではなく明示的な結果型を優先する。
- 正規化が必要なら、正規化と検証の順序を業務規則として固定する。
- 単位が異なる値を、同じ数値型として交換可能にしない。
- 型アサーション、未検証の型変換、汎用的な復元処理による迂回を許可しない。
- 不変条件の異なる値を、再利用だけを目的として一つの型にまとめない。
- スキーマ検証ライブラリを使う場合も、検証を通った値と未検証の値を
  別の型として区別する。検証関数を呼んだかどうかを、呼び出し規約で管理しない。
- ライブラリが検証を迂回できる経路を持つ場合は、その経路を使用禁止にする。
  zodでは型表明（`as`）、Pydanticでは`model_construct`が該当する。
- Pydanticのモデルは既定で可変である。作成後の変更を禁止する設定を明示する。

## TypeScriptによる例

```ts
type Result<T, E> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

type UnitQuantityError = "not-an-integer" | "out-of-range";

export class UnitQuantity {
  private constructor(readonly value: number) {}

  static create(value: number): Result<UnitQuantity, UnitQuantityError> {
    if (!Number.isInteger(value)) {
      return { ok: false, error: "not-an-integer" };
    }

    if (value < 1 || value > 1000) {
      return { ok: false, error: "out-of-range" };
    }

    return { ok: true, value: new UnitQuantity(value) };
  }
}
```

`UnitQuantity`を受け取った内部処理では、1以上1000以下の整数かを再検証しない。外部入力を受け取る境界では、必ず`UnitQuantity.create`を通す。

## zodによる例

`brand`によって、同じ`string`や`number`から作られた値を型として区別する。
`safeParse`は成功と失敗を判別可能な結果として返すため、失敗の処理を強制できる。

```ts
const UnitQuantity = z
  .number()
  .int()
  .min(1)
  .max(1000)
  .brand<"UnitQuantity">();

type UnitQuantity = z.infer<typeof UnitQuantity>;

const parsed = UnitQuantity.safeParse(input);
if (!parsed.success) {
  // 失敗の処理を書かなければ値を取り出せない
}
```

`brand`は名義的な区別を与えるが、コンストラクタの非公開化と同等ではない。
`input as UnitQuantity`と書けば検証を迂回できる。生成をモジュール境界で絞り、
型表明を静的解析規則で禁止する。

失敗を例外として扱う`parse`ではなく、結果を返す`safeParse`を既定とする。
`parse`を使う場合は、境界の外側で例外を結果型へ変換する。

## Pydanticによる例

`Annotated`と制約で不変条件を表す。モデルの作成時に検証が実行されるため、
モデルの生成自体がスマートコンストラクタとして機能する。

```python
from typing import Annotated
from pydantic import BaseModel, ConfigDict, Field

UnitQuantity = Annotated[int, Field(ge=1, le=1000)]

class OrderLine(BaseModel):
    model_config = ConfigDict(frozen=True)

    quantity: UnitQuantity
```

注意する点が三つある。

- `frozen=True`を指定しないと、作成後に値を変更できる。
- `model_construct`は検証を実行しない。使用しない。
- 検証の失敗は`ValidationError`として送出される。想定内の失敗として扱う場合は、
  境界で捕捉し、結果型または明示的な戻り値へ変換する。

`Annotated`の別名だけでは、意味の異なる値を型として区別できない。
`UnitQuantity`と`KilogramQuantity`をどちらも`Annotated[int, ...]`にすると
交換可能になる。区別が必要な値は、別のクラスとして定義する。

## レビュー項目

- 不正な値を公開コンストラクタから生成できないか。
- 重要な値がプリミティブ型だけで表現されていないか。
- 同じ型に、異なる意味や単位の値が混在していないか。
- 検証失敗が無視される戻り値になっていないか。
- 作成後に内部値を変更できないか。
- 型アサーションによってスマートコンストラクタを迂回していないか。
- データベースからの復元時に、未検証の値を検証済みとして扱っていないか。
- 不変条件を変更した際に、保存済みデータの移行または再検証が必要か。
- 型表明または`model_construct`によって検証を迂回していないか。
- Pydanticのモデルに、作成後の変更を禁止する設定があるか。
- スキーマの定義と、ドメインで使う型が同一の定義から導かれているか。
- 検証の失敗が例外として送出される場合、境界で結果型へ変換しているか。

## 型だけでは保証できない範囲

現在日時、外部サービス、他の集約の状態などに依存する規則は、単一値の型だけでは保証しない。必要な依存情報を受け取る検証処理またはワークフローで保証する。
