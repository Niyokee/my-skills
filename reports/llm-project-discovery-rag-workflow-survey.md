# LLM/RAGプロジェクトの進め方：実データ収集・既存アプリ検証・合成データ評価の実践ガイド

- 調査日: 2026年9月17日
- 対象: 大規模言語モデル（LLM）と検索拡張生成（RAG）を用いるプロジェクトの初期検証
- 中心となる問い: 「実データを集め、既存アプリで試し、合成データで評価してから独自実装へ進む」という進め方は、公開されている実務ガイドや研究と合致するか

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [3つのステップに対する検証と実践ポイント](#2-3つのステップに対する検証と実践ポイント)
   1. [検索対象データを集める](#21-検索対象データを集める)
   2. [NotebookLMなどの既存アプリで試す](#22-notebooklmなどの既存アプリで試す)
   3. [合成データを作ってLLMを動かす](#23-合成データを作ってllmを動かす)
3. [実務向け標準開発フロー](#3-実務向け標準開発フロー)
   1. [Step 0: 対象業務と判断基準を定義する](#step-0-対象業務と判断基準を定義する)
   2. [Step 1: 代表的な実データと実際の問いを集める](#step-1-代表的な実データと実際の問いを集める)
   3. [Step 2: 既存アプリで価値と情報源を検証する](#step-2-既存アプリで価値と情報源を検証する)
   4. [Step 3: 評価データセットを作成する](#step-3-評価データセットを作成する)
   5. [Step 4: 最小構成で独自のLLM/RAGを動かす](#step-4-最小構成で独自のllmragを動かす)
   6. [Step 5: 小規模に利用し、実ログへ置き換える](#step-5-小規模に利用し実ログへ置き換える)
4. [実務で注意すべき4つの落とし穴](#4-実務で注意すべき4つの落とし穴)
5. [まとめ](#5-まとめ)
6. [調査上の限界](#6-調査上の限界)
7. [出典一覧](#7-出典一覧)

## 1. エグゼクティブサマリー

「**実データを集め、NotebookLMなどの既存アプリで試し、合成データを作ってLLMを動かす**」という初期アプローチは、公開されている実務ガイドや研究に照らしても、**基本的に極めて妥当**である。

完成品を一から作り込まず、まずは手元にある実データと既存ツールを使って価値を素早く検証する姿勢は、手戻りを減らす上で有効である。ただし、組織の標準プロセスとして定着させるには、次の3点を補う必要がある。

| 元のステップ | 評価 | 実務で補うべきポイント |
| --- | --- | --- |
| **1. 検索対象データを集める** | ◎ 非常に妥当 | 単に文書を集めるのではなく、先に**対象業務、業務要件、実際の質問例**をセットで定義する。 |
| **2. 既存アプリで試す** | ◯ 有効 | 検証できるのは主に**回答の有用性と元データの十分性**である。独自RAGの検索精度、費用、応答時間、権限制御までは保証しない。 |
| **3. 合成データでLLMを動かす** | ◯ 条件付きで有効 | 合成データは主に**評価用の質問と期待回答**として使い、実例で不足する範囲を補う。 |

調査した公開資料は、元の3ステップをそのまま一つの標準手順として規定してはいない。しかし、業務要件の定義、代表的な実データと質問の収集、既存ツールでの価値検証、合成評価データの利用、最小構成からの反復という各要素は、Microsoft、Google、OpenAI、AWS、Anthropicの実務資料と、ARES・RAGASの研究によって個別に支持されている。

## 2. 3つのステップに対する検証と実践ポイント

### 2.1 検索対象データを集める

#### データの収集前に「業務要件」と「問い」を定義する

MicrosoftのRAG準備ガイドは、最初に対象業務と要件を明確にするよう求めている。

```text
原文: Clearly define the business requirements for the RAG solution.
日本語訳: RAGソリューションに対する業務要件を明確に定義する。
```

[Microsoft, 2025/10, RAG solution design and evaluation preparation](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag)

また、代表的な文書を集める作業と並行して、利用者が実際に尋ねる質問も集めるよう推奨している。

```text
原文: Do this step while you gather the representative content.
日本語訳: 代表的なコンテンツを集める作業と並行して、このステップを実施する。
```

[Microsoft, 2025/10, RAG solution design and evaluation preparation](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag)

文書だけを蓄積しても、想定される質問への答えが含まれていなければRAGは機能しない。したがって、「文書」「質問」「期待回答」「根拠箇所」を可能な範囲でセットにして収集するのが実務上の出発点になる。

#### 検索対象には合成文書より実データを優先する

検索対象となる文書は、AIが生成した合成文書より、現場で実際に使われている文書を優先する。Microsoftも、評価用の代表的なコンテンツには実コンテンツを選ぶよう明記している。

```text
原文: Choose real content over synthetic content.
日本語訳: 合成コンテンツではなく、実際のコンテンツを選ぶ。
```

[Microsoft, 2025/10, RAG solution design and evaluation preparation](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag)

実データには、業務固有の表記ゆれ、古い記述、矛盾、欠落、形式の違いが含まれる。これらは本番で直面する条件そのものであり、価値検証の段階から観察対象に含める必要がある。

### 2.2 NotebookLMなどの既存アプリで試す

#### 「情報源に基づく回答の価値」を素早く確認する

NotebookLMは2026年7月にGemini Notebookへ名称変更された。以下では、一般に定着している旧称を含めて「NotebookLMなど」と表記する。[Google, 2026/07, NotebookLM is now Gemini Notebook](https://blog.google/technology/google-labs/notebooklm-is-now-gemini-notebook/)

Gemini Notebookの回答は、登録した情報源に基づくよう設計されている。

```text
原文: Responses are grounded exclusively in your notebook sources.
日本語訳: 回答は、そのノートブックに登録された情報源だけを根拠とする。
```

[Google, 2026/09, Use chat in Gemini Notebook](https://support.google.com/gemini-notebook/answer/16206563)

そのため、「手元の文書だけで業務上有用な回答が成立するか」「引用や根拠を確認しながら回答を使えるか」「不足・陳腐化・矛盾している情報は何か」を、独自実装の前に確認できる。

OpenAIも、企業内の知識を使う初期手段としてCustom GPTを挙げている。

```text
原文: Upload files to give the GPT domain knowledge.
日本語訳: GPTに対象領域の知識を与えるため、ファイルをアップロードする。
```

[OpenAI, 2026/09, Creating a GPT](https://help.openai.com/en/articles/8554397-creating-a-gpt)

#### 既存アプリで「分かること」と「分からないこと」を区別する

既存アプリで分かることは、主に次の3点である。

- 手元の文書だけで求める回答が成立するか
- 根拠や引用付きの回答が業務に役立つか
- 足りない文書、古い文書、矛盾している文書は何か

一方、独自実装に移るまで分からないこともある。

- 文書の分割方法や検索方式を含む検索精度
- 利用者の権限に応じた情報の出し分け
- 応答時間、運用費用、可用性、監査性
- 独自システムで採用するモデルや検索基盤による回答品質

したがって、既存アプリでの検証は「価値仮説と情報源の検証」であり、独自RAGの性能検証そのものではない。独自実装では、検索と回答生成を分けて測る必要がある。Microsoftも、RAGの各段階を独立して評価するよう推奨している。

```text
原文: You should evaluate each step independently for optimization.
日本語訳: 最適化のため、各ステップを独立して評価するべきである。
```

[Microsoft, 2025/05, Design and develop a RAG solution](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)

### 2.3 合成データを作ってLLMを動かす

#### 合成データを「評価用のテストデータ」として使う

初期段階では、本番利用者の質問ログが十分に存在しない。そこで、実際の文書から質問と期待回答を作り、評価データセットを補う方法が有効になる。AWSは、実利用の質問を評価データへ含めることを重視しつつ、RAG向けの合成質問生成手順も紹介している。

```text
原文: Include actual user questions in your evaluation dataset.
日本語訳: 評価データセットには、実際の利用者の質問を含める。
```

[AWS, 2024/03, Generate synthetic data for evaluating RAG systems](https://aws.amazon.com/blogs/machine-learning/generate-synthetic-data-for-evaluating-rag-systems-using-amazon-bedrock/)

合成データは、実データを置き換えるものではない。実例を核にし、言い換え、例外、答えが存在しない質問、境界条件など、実例だけでは不足する範囲を補うために使う。生成後は、対象業務を知る担当者が質問、期待回答、根拠箇所の妥当性を確認する。

#### 研究上の位置づけ

ARESは、合成データと少量の人手注釈を組み合わせ、RAGシステムを評価する枠組みである。

```text
原文: ARES leverages synthetic data and a small set of human annotations.
日本語訳: ARESは、合成データと少量の人手注釈を活用する。
```

[Saad-Falconほか, 2024/06, ARES](https://aclanthology.org/2024.naacl-long.20/)

ARESはNAACL 2024の論文で、Stanford University、University of Illinois Urbana-Champaign、University of California, Berkeley、University of Washingtonなどの研究者による。OpenAlexでは2026年9月17日時点で123件の被引用数が確認できた。[OpenAlex, 2026/09, ARES](https://openalex.org/W4387170227)

RAGASは、検索された文脈の妥当性、回答の忠実性、回答の関連性などを分けて評価する枠組みである。合成データ生成そのものの根拠というより、RAGの評価軸を分離する根拠として位置づけるのが正確である。

```text
原文: We formulate three quality aspects: faithfulness, answer relevance, and context relevance.
日本語訳: 忠実性、回答の関連性、文脈の関連性という3つの品質側面を定式化する。
```

[Esほか, 2024/03, RAGAS](https://aclanthology.org/2024.eacl-demo.16/)

RAGASはEACL 2024の論文で、Exploding Gradients、University of Copenhagen、Amazon Alexa AIなどの著者による。OpenAlexでは2026年9月17日時点で456件の被引用数が確認できた。[OpenAlex, 2026/09, RAGAS](https://openalex.org/W4380237916)

#### 最初から大量に作らず、少数から始める

Anthropicは、初期の評価セットとして20〜50件程度の単純な課題を出発点にできるとしている。

```text
原文: Even 20–50 simple tasks drawn from real failures is a great start.
日本語訳: 実際の失敗から選んだ20〜50件の単純な課題でも、良い出発点になる。
```

[Anthropic, 2026/01, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

ここで重要なのは、20〜50件で評価が完成するという意味ではなく、少数の実例から評価を始め、失敗ログを取り込んで継続的に更新するという点である。

## 3. 実務向け標準開発フロー

以上を統合すると、組織で再利用しやすい標準フローは次の6ステップになる。これは単一の資料に記載された手順ではなく、本調査で確認した実務ガイドと研究を統合した提案である。

```text
[Step 0] 業務要件の定義
   ↓
[Step 1] 実データと「問い」の収集
   ↓
[Step 2] 既存アプリで価値検証
   ↓
[Step 3] 評価データセットの作成（実例＋合成データ）
   ↓
[Step 4] 最小構成で独自のLLM/RAGを構築・測定
   ↓
[Step 5] 小規模運用と実ログへの置き換え
```

### Step 0: 対象業務と判断基準を定義する

- 「誰が、どの業務で、何を達成するための仕組みか」を明文化する。
- 回答の正しさ、根拠との一致、業務時間、応答時間、費用など、成功とみなす基準を定める。
- 誤情報、情報漏えい、根拠のない断定など、許容できない失敗を定める。

評価を先に定義することで、要件を実装可能で検証可能な形へ変換できる。Anthropicも、評価駆動開発は製品要件を明確な成功基準へ落とし込むと説明している。[Anthropic, 2026/01, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### Step 1: 代表的な実データと実際の問いを集める

- よく参照される公式文書だけでなく、形式の異なる文書、古い文書、矛盾を含む文書も対象にする。
- 過去の問い合わせ履歴や担当者へのヒアリングから、実際の質問例を集める。
- 可能であれば「質問」「期待回答」「根拠箇所」「重要度」をセットで記録する。

### Step 2: 既存アプリで価値と情報源を検証する

- 組織の規程上、登録を許可された文書だけをGemini NotebookやCustom GPTなどへ登録する。
- Step 1で集めた質問を入力し、対象業務の担当者が回答と根拠を確認する。
- 「このデータで業務が成立するか」「不足・陳腐化・矛盾している情報は何か」を判定する。
- この段階の結果は、独自RAGの性能保証ではなく、価値仮説と情報源の検証結果として記録する。

### Step 3: 評価データセットを作成する

- Step 1〜2で得た実例を評価データセットの核にする。
- 不足する言い換え、例外、答えがない質問、境界条件を合成データで補う。
- 対象業務の担当者が、質問、期待回答、根拠箇所、採点基準を確認する。
- 実例と合成例を区別して記録し、後から構成比と成績を確認できるようにする。

### Step 4: 最小構成で独自のLLM/RAGを動かす

- 最初からマルチエージェントや複雑な検索処理を導入せず、単純なプロンプトと標準的な検索構成から始める。
- 最初の比較対象となる構成を、基準構成（ベースライン）として固定する。
- 「必要な文書を検索できたか」と「取得した文書に基づいて正しく回答できたか」を分けて測る。
- 品質だけでなく、応答時間、費用、失敗率も同じ条件で記録する。

Anthropicも、複雑さは成果を明確に改善すると確認できた場合に追加し、まず最も単純な解決策を探すよう勧めている。[Anthropic, 2024/12, Building effective agents](https://www.anthropic.com/research/building-effective-agents)

### Step 5: 小規模に利用し、実ログへ置き換える

- 限定した利用者と対象業務で試験運用する。
- 利用者の許可と組織のデータ取扱規程の範囲で、実際の質問、回答、評価、失敗を記録する。
- 実ログから新しい評価例を追加し、合成例を少しずつ実例へ置き換える。
- 同じ評価データセットで基準構成と変更後の構成を比較し、改善と悪化の両方を確認する。

この段階では、失敗を発見して評価例へ変換する速度が重要になる。評価セットは一度作って終わりではなく、利用実態とともに更新する運用資産である。[Hamel Husain, 2024/03, Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/)

## 4. 実務で注意すべき4つの落とし穴

### 4.1 「既存アプリでの成功＝独自RAGの成功」ではない

既存製品の内部構成は、独自実装で採用するモデル、検索基盤、分割方法、権限制御と同一ではない。したがって、既存アプリで良い結果が出たことから「価値がありそう」「情報源が足りそう」とはいえるが、独自RAGでも同じ品質が出るとは断定できない。

### 4.2 合成データだけでは利用頻度を測れない

合成データは、想定する例外や難しい条件を意図的に増やす用途には向く。しかし、現場でどの質問がどれだけ発生するかという頻度分布は再現できない。Anthropicも、合成評価は実利用者の微妙な挙動を取りこぼす可能性があると指摘している。[Anthropic, 2026/01, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### 4.3 合成データを早く大量に作りすぎない

要件が固まらない段階で大量生成すると、現場の実態と離れた質問や、生成モデルに都合のよい期待回答が増える恐れがある。少数の実例から始め、専門家の確認を通し、実ログが増えるたびに評価データセットを更新する。

### 4.4 機密情報の取扱規程を確認する

既存アプリで試す前に、社内データの登録可否、サービス提供者によるデータ利用、保持期間、保存地域、削除方法、利用者の権限を確認する。Googleも、Gemini Notebookの利用時には組織の管理設定とプライバシー条件を確認するよう案内している。[Google, 2026/09, Gemini Notebook privacy and data handling](https://support.google.com/gemini-notebook/answer/16164461)

## 5. まとめ

提示された進め方は、**実装に着手する前に、データと既存ツールで素早く当たりをつける**という点で、実務上よく設計されている。

組織の標準スキルとして定義するなら、基本方針は次のようにまとめられる。

**システムの実装から始めず、対象業務、現場の実データ、実際の問い、評価基準の定義から始める。既存アプリで価値とデータの十分性を検証し、実例を核として合成データで不足を補いながら、最小構成から測定を始める。運用開始後は、合成例を実ログへ置き換えて評価データセットを育てる。**

元の3ステップは削るべきではない。前段に「業務要件と評価基準の定義」を加え、後段に「独自実装の分離評価」と「小規模運用から得た実ログによる更新」を加えることで、再現可能な6ステップの標準プロセスになる。

## 6. 調査上の限界

- 元の3ステップ全体を一つの工程として比較検証した研究は確認できなかった。本レポートの6ステップは、複数の実務資料と研究を統合した提案である。
- NotebookLMなどでの成功が、独自RAGの成功をどの程度予測するかを定量的に示した比較研究は確認できなかった。
- 実務資料の多くは製品提供者による。自社製品に有利な前提が含まれる可能性があるため、個別製品の推奨ではなく、複数資料に共通する工程を抽出した。
- ARESとRAGASは、RAG評価の個別手法を支える研究であり、LLMプロジェクト全体の進行方法を直接検証した研究ではない。
- 被引用数はOpenAlex上の2026年9月17日時点の値であり、今後変動する。

## 7. 出典一覧

### 公式資料・実務ガイド

- [Microsoft, 2025/10] Microsoft. “RAG solution design and evaluation preparation.” Microsoft Learn. https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag
- [Microsoft, 2025/05] Microsoft. “Design and develop a RAG solution.” Microsoft Learn. https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide
- [Google, 2026/07] Google. “NotebookLM is now Gemini Notebook.” Google Blog. https://blog.google/technology/google-labs/notebooklm-is-now-gemini-notebook/
- [Google, 2026/09] Google. “Use chat in Gemini Notebook.” Google Help. https://support.google.com/gemini-notebook/answer/16206563
- [Google, 2026/09] Google. “Gemini Notebook privacy and data handling.” Google Help. https://support.google.com/gemini-notebook/answer/16164461
- [OpenAI, 2026/09] OpenAI. “Creating a GPT.” OpenAI Help Center. https://help.openai.com/en/articles/8554397-creating-a-gpt
- [AWS, 2024/03] Amazon Web Services. “Generate synthetic data for evaluating RAG systems using Amazon Bedrock.” AWS Machine Learning Blog. https://aws.amazon.com/blogs/machine-learning/generate-synthetic-data-for-evaluating-rag-systems-using-amazon-bedrock/
- [Anthropic, 2024/12] Anthropic. “Building effective agents.” Anthropic Research. https://www.anthropic.com/research/building-effective-agents
- [Anthropic, 2026/01] Anthropic. “Demystifying evals for AI agents.” Anthropic Engineering. https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- [Hamel Husain, 2024/03] Hamel Husain. “Your AI Product Needs Evals.” hamel.dev. https://hamel.dev/blog/posts/evals/

### 学術論文

- [Saad-Falconほか, 2024/06] Saad-Falcon, J. et al. “ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems.” *Proceedings of NAACL 2024*. https://aclanthology.org/2024.naacl-long.20/
- [Esほか, 2024/03] Es, S. et al. “RAGAS: Automated Evaluation of Retrieval Augmented Generation.” *Proceedings of EACL 2024: System Demonstrations*. https://aclanthology.org/2024.eacl-demo.16/

### 論文メタデータ

- [OpenAlex, 2026/09] “ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems.” https://openalex.org/W4387170227
- [OpenAlex, 2026/09] “RAGAS: Automated Evaluation of Retrieval Augmented Generation.” https://openalex.org/W4380237916
