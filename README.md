# my-skills

Codex で要件定義とドメイン設計を支援する、個人用のスキル集です。

このリポジトリには、RDRA（Relationship Driven Requirement Analysis）による要件モデリング、業務規則を値・状態・ワークフローの型で保証するための設計指針、集約境界・整合性・外部境界の設計指針を収録しています。

## 収録スキル

| スキル | 配置場所 | 用途 |
| --- | --- | --- |
| `rdra` | [`spec-guidelines/rdra`](spec-guidelines/rdra) | 要求、業務、ユースケース、情報、状態の関係を整理し、要件の抜け漏れや矛盾を確認します。 |
| `type-guidelines` | [`type-guidelines`](type-guidelines) | 業務上の不変条件を、制約付きの値、状態、集約型、ワークフローの型として設計またはレビューします。 |
| `design-guidelines` | [`design-guidelines`](design-guidelines) | 集約境界、更新責任、トランザクション、結果整合性、ドメインと外部との境界を設計またはレビューします。 |

3種類のスキルは、次の範囲を担当します。

1. `rdra` が、システムに必要な価値、業務、ユースケース、情報、状態を明らかにします。
2. `type-guidelines` が、明らかになった業務規則のうち、値、状態、集約、ワークフローの型で保証できる範囲を扱います。
3. `design-guidelines` が、永続化、外部との変換、複数集約の連携を含む、実行時の整合性を扱います。

## 使い方

プロンプトでスキル名を指定します。

```text
$rdra を使って、受注管理システムの要件を整理してください。
```

```text
$type-guidelines を使って、注文ドメインの型設計をレビューしてください。
```

```text
$design-guidelines を使って、注文と請求の集約境界と整合性をレビューしてください。
```

スキル名を明示しない場合でも、依頼内容が各スキルの適用条件と一致すれば、Codex がスキルを選択することがあります。

## ディレクトリ構成

```text
my-skills/
├── design-guidelines/
│   ├── SKILL.md
│   ├── agents/openai.yaml
│   └── references/
├── spec-guidelines/
│   └── rdra/
│       ├── SKILL.md
│       ├── agents/openai.yaml
│       └── references/
└── type-guidelines/
    ├── SKILL.md
    ├── agents/openai.yaml
    └── references/
```

- `SKILL.md` は、スキルの適用条件、作業手順、出力要件を定義します。
- `references/` は、作業内容に応じて参照する詳細なガイドラインを格納します。
- `agents/openai.yaml` は、Codex 上の表示名、説明、既定のプロンプトを定義します。

## スキルを更新する場合

スキルを変更するときは、次の対応関係を保ちます。

- `SKILL.md` の `description` には、スキルを使用する条件を具体的に記載します。
- 詳細な判断基準は `references/` に分け、`SKILL.md` から参照する条件を明記します。
- `agents/openai.yaml` の表示内容は、`SKILL.md` の目的と一致させます。
- レビュー用のスキルでは、依頼がレビューだけの場合に成果物を変更しないことを明記します。
