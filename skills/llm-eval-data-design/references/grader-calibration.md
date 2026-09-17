# 参照解と採点器の設計

## 採点方法の優先順位

一つの採点器へすべての判断を任せず、判定内容に応じて使い分ける。

1. 決定的な規則
2. 人間による評価
3. LLMによる採点

OpenAIは、比較、分類、基準に基づく採点を、自由記述の生成品質を直接採点する方法より信頼しやすい形式として挙げている。LLMによる採点は、人間の判断との一致を確認してから使う。[^openai-best-practices]

## 決定的な規則

次のような項目に使う。

- JSONやスキーマへの適合
- 必須項目の有無
- 数値計算の結果
- ツールまたはAPIの実行結果
- 許可されていないツールの使用
- 引用した識別子やURLの存在
- 状態遷移または完了条件

規則で判定できる項目を、LLMによる採点へ置き換えない。

## 参照解と採点基準

- 一意の正解がある場合は、正解と根拠を保存する。
- 複数の正解を許容する場合は、許容条件を列挙する。
- 絶対に含めてはいけない内容は、禁止条件として分ける。
- 段階評価では、各段階を観測可能な条件で定義する。
- 最終回答だけでなく過程を評価する場合は、行動、引数、状態変化を別々に採点する。
- 二人の専門家が独立に判定できない場合は、ケースまたは基準を修正する。

Anthropicは、課題が明確で、二人の専門家が同じ合否判断を出せることと、参照解を用意することを評価設計の条件として挙げている。[^anthropic-evals]

## LLMによる採点の校正

### 校正集合を作る

- 人間が独立に判定した事例を使う。
- 合格と不合格の両方を含める。
- 境界事例と重大な失敗を含める。
- 採点指示の開発に使う集合と、最終確認に使う集合を分ける。

### 確認する結果

- 混同行列
- 合格事例に対する一致率
- 不合格事例に対する一致率
- 判定不能率
- 層別の不一致
- 不一致事例の内容
- 指標の信頼区間

一致は正しさそのものではない。FDAは、評価者間の一致度と、正解に対する正しさを区別している。[^fda-agreement]

### 偏りを確認する

LLMによる採点には、回答の提示順序、冗長さ、自己生成回答への選好、推論能力の限界がある。[^mt-bench] 次を確認する。

- 回答の順序を入れ替える。
- 内容を保ったまま長さを変える。
- 参照解の有無を変える。
- 評価対象と異なるモデル系列の採点器を試す。
- 人間との不一致が多い層を特定する。

## 評価基準の変化

出力を観察すると、人間が重視する基準自体が変わる場合がある。EvalGenは、評価出力の観察を通じて基準が変化する現象を扱っている。[^evalgen] 基準を変えた場合は、採点器だけでなく過去の評価結果との比較可能性も見直す。

## 高リスクケース

- 対象分野の専門家が参照解と判定を確認する。
- LLMによる採点だけでリリース可否を決めない。
- 不一致を平均値に埋めず、事例単位で確認する。
- 誤った参照解が引き起こす影響と、発生可能性に応じて確認率を上げる。[^aws-ground-truth]

## 完了条件

- 各評価項目に採点方法が割り当てられている。
- 参照解または採点基準の作成者と確認者が分かる。
- LLMによる採点を、人間が判定した未使用事例で確認している。
- 合格と不合格の両方で性能を確認している。
- 順序と長さによる偏りを確認している。
- 高リスクケースに人間の確認が残っている。

[^openai-best-practices]: OpenAI. “Evaluation best practices.” OpenAI Developers, 2026-09-17閲覧. https://developers.openai.com/api/docs/guides/evaluation-best-practices
[^anthropic-evals]: Mikaela Grace, Jeremy Hadfield, Rodrigo Olivares, Jiri De Jonghe. “Demystifying evals for AI agents.” Anthropic Engineering, 2026-01-09. https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
[^fda-agreement]: U.S. Food and Drug Administration. “Statistical Guidance on Reporting Results from Studies Evaluating Diagnostic Tests.” 2007-03-13. https://www.fda.gov/regulatory-information/search-fda-guidance-documents/statistical-guidance-reporting-results-studies-evaluating-diagnostic-tests-guidance-industry-and-fda
[^mt-bench]: Lianmin Zheng et al. “Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.” NeurIPS 2023 Datasets and Benchmarks. https://proceedings.neurips.cc/paper_files/paper/2023/hash/91f18a1287b398d378ef22505bf41832-Abstract-Datasets_and_Benchmarks.html
[^evalgen]: Shreya Shankar, J.D. Zamfirescu-Pereira, Björn Hartmann, Aditya G. Parameswaran, Ian Arawjo. “Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences.” UIST 2024. https://arxiv.org/abs/2404.12272
[^aws-ground-truth]: Samantha Stuart, Ivan Cui, Philippe Duplessis-Guindon, Rahul Jani. “Ground truth generation and review best practices for evaluating generative AI question answering with FMEval.” AWS Machine Learning Blog, 2025-03-05. https://aws.amazon.com/blogs/machine-learning/ground-truth-generation-and-review-best-practices-for-evaluating-generative-ai-question-answering-with-fmeval/
