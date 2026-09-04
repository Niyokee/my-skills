# my-skills

Agent Skills（AIエージェントに作業手順と参照資料を追加する共通形式）に対応するAIエージェントで、要件定義とドメイン設計を支援するスキル集です。

このリポジトリには、RDRA（Relationship Driven Requirement Analysis）による要件モデリング、対話型データモデリング、業務規則を値・状態・ワークフローの型で保証するための設計指針、集約境界・整合性・外部境界の設計指針を収録しています。

## インストール

Node.jsが利用できる端末で、[`skills` コマンドラインインターフェース（CLI）](https://github.com/vercel-labs/skills)を実行します。インストールするスキルと、Claude Code、Codex、Cursorなどの利用先を対話形式で選択できます。

```bash
npx skills add Niyokee/my-skills -g
```

`-g`は、選択したスキルを利用者単位でインストールし、そのマシンの複数のプロジェクトから使えるようにする指定です。

インストール済みのスキルは、次のコマンドで確認できます。

```bash
npx skills list -g
```

GitHubに公開された最新版へ更新する場合は、次のコマンドを実行します。

```bash
npx skills update rdra data-modeling-guidelines type-guidelines design-guidelines -g
```

## 収録スキル

| スキル | 配置場所 | 用途 |
| --- | --- | --- |
| `rdra` | [`skills/rdra`](skills/rdra) | 要求、業務、ユースケース、情報、状態の関係を整理し、要件の抜け漏れや矛盾を確認します。 |
| `data-modeling-guidelines` | [`skills/data-modeling-guidelines`](skills/data-modeling-guidelines) | 関係者への対話から概念データモデルを作り、データベース設計への変換、レビュー、設計指針の策定を行います。 |
| `type-guidelines` | [`skills/type-guidelines`](skills/type-guidelines) | 業務上の不変条件を、制約付きの値、状態、集約型、ワークフローの型として設計またはレビューします。 |
| `design-guidelines` | [`skills/design-guidelines`](skills/design-guidelines) | 集約境界、更新責任、トランザクション、結果整合性、ドメインと外部との境界を設計またはレビューします。 |

4種類のスキルは、次の範囲を担当します。

1. `rdra` が、システムに必要な価値、業務、ユースケース、情報、状態を明らかにします。
2. `data-modeling-guidelines` が、業務上の情報構造と制約を対話で明らかにし、概念データモデルから論理・物理データベース設計へ変換します。
3. `type-guidelines` が、明らかになった業務規則のうち、値、状態、集約、ワークフローの型で保証できる範囲を扱います。
4. `design-guidelines` が、永続化、外部との変換、複数集約の連携を含む、実行時の整合性を扱います。

## 使い方

依頼内容が各スキルの適用条件と一致すると、対応するAIエージェントがスキルを選択します。

```text
受注管理システムの要件をRDRAで整理してください。
```

スキル名を明示する方法は、利用するAIエージェントによって異なります。Codexでは、スキル名の先頭に`$`を付けます。

```text
$rdra を使って、受注管理システムの要件を整理してください。
```

```text
$data-modeling-guidelines を使って、受注業務の具体例から概念データモデルを作成してください。
```

```text
$type-guidelines を使って、注文ドメインの型設計をレビューしてください。
```

```text
$design-guidelines を使って、注文と請求の集約境界と整合性をレビューしてください。
```

Claude Codeでは、スラッシュコマンドとして指定します。

```text
/rdra 受注管理システムの要件を整理してください。
```

## ディレクトリ構成

```text
my-skills/
└── skills/
    ├── design-guidelines/
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   └── references/
    ├── data-modeling-guidelines/
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   └── references/
    ├── rdra/
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   └── references/
    └── type-guidelines/
        ├── SKILL.md
        ├── agents/openai.yaml
        └── references/
```

- `SKILL.md` は、スキルの適用条件、作業手順、出力要件を定義します。
- `references/` は、作業内容に応じて参照する詳細なガイドラインを格納します。
- `agents/openai.yaml` は、OpenAI製品上の表示名、説明、既定のプロンプトを定義します。他のAIエージェントは、このファイルを使用しない場合があります。

## スキルを更新する場合

スキルを変更するときは、次の対応関係を保ちます。

- `SKILL.md` の `description` には、スキルを使用する条件を具体的に記載します。
- 詳細な判断基準は `references/` に分け、`SKILL.md` から参照する条件を明記します。
- `agents/openai.yaml` の表示内容は、`SKILL.md` の目的と一致させます。
- レビュー用のスキルでは、依頼がレビューだけの場合に成果物を変更しないことを明記します。
