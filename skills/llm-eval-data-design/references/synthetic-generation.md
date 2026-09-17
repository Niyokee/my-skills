# 合成評価データの生成と品質確認

## 目次

1. [生成前の準備](#生成前の準備)
2. [生成手順](#生成手順)
3. [RAGで保存する項目](#ragで保存する項目)
4. [エージェントで保存する項目](#エージェントで保存する項目)
5. [品質軸](#品質軸)
6. [記録する生成条件](#記録する生成条件)

## 生成前の準備

生成を始める前に、次を定義する。

- 評価対象となる業務フロー
- 期待する挙動
- 失敗分類と重大度
- 利用可能な情報と禁止された情報
- 参照解または採点基準
- 一件として保存する入力、文脈、出力、実行履歴

OpenAIは、評価データに典型例、境界例、敵対的事例を含め、人間の専門家によるラベルを使うよう勧めている。[^openai-best-practices]

## 生成手順

### 1. 人間が種ケースを書く

製品要件、手動試験、対象分野の専門家が持つ失敗仮説から、少数の種ケースを作る。典型例だけでなく、拒否、確認、情報不足、境界条件を含める。

### 2. 属性の組を作る

業務フロー、利用者、重大度、期待する挙動、難易度、入力形式などの値を組み合わせる。先に構造化した属性の組を作り、その後に自然な文章へ変換する。HusainとShankarは、この二段階方式を、無構造な一括生成よりカバレッジを制御しやすい方法として紹介している。[^husain-synthetic]

### 3. 自然な入力へ具体化する

意味を保ちながら、語調、長さ、言語、誤字、文脈、利用者像を変える。表層的な言い換えだけを増やさない。

### 4. 実際のアプリケーションを実行する

モデル単体の回答ではなく、検索、ツール、権限、メモリ、外部依存先を含む実際の構成を使う。最終回答だけでなく、中間結果と実行履歴を保存する。

### 5. 参照解と根拠を作る

一意の正解がある場合は、正解と根拠を保存する。一意の正解がない場合は、許容条件、禁止条件、段階別の採点基準を作る。

### 6. 選別する

次のケースを除外または修正する。

- 対象製品では成立しない
- 必要な情報がなく解答不能
- 期待する挙動が曖昧
- 既存ケースと意味的に重複する
- 入力内に正解または採点基準が漏れている
- 本番では利用できない権限や機能を前提にしている

AWSのRAG向け手順は、質問、回答、根拠箇所、利用スタイルを生成し、対象分野の専門家または批評用モデルで選別する。[^aws-rag] AWSは、LLMが作った正解データを対象分野の専門家の代替として扱わないよう求めている。[^aws-ground-truth]

## RAGで保存する項目

検索拡張生成（RAG）では、次を保存する。

- 質問
- 期待回答
- 回答を支える文書と該当箇所
- 必要な文書が検索候補に存在するか
- 実際に検索された文書
- 最終回答が参照した根拠

検索失敗と回答生成の失敗を分けて判定する。

## エージェントで保存する項目

- 目的と完了条件
- 利用可能なツールと権限
- 期待する行動または禁止する行動
- 選択したツールと引数
- 行動順序
- 失敗時の回復処理
- 停止条件
- 最終状態

## 品質軸

- 妥当性: 対象製品で成立するか
- 解答可能性: 与えた情報と機能で解けるか
- 正解の信頼性: 専門家が同じ判定を出せるか
- カバレッジ: 重要な業務、重大リスク、期待する挙動を覆うか
- 多様性: 意味、構造、文体、難易度、実行経路が異なるか
- 忠実度: 想定する利用者、業務、文書の制約を保つか
- 漏洩: 正解や採点規則が入力へ混ざっていないか
- 再現性: 生成条件と版を追跡できるか

合成テキストの多様性が実データより低い傾向を報告した研究があるため、生成件数だけで品質を判断しない。[^li-synthetic]

## 記録する生成条件

- 生成モデルと版
- 生成指示
- 参照した種ケースと資料
- 乱数種と生成設定
- 生成日時
- 適用した層
- 自動選別と人間確認の結果
- 修正履歴

[^openai-best-practices]: OpenAI. “Evaluation best practices.” OpenAI Developers, 2026-09-17閲覧. https://developers.openai.com/api/docs/guides/evaluation-best-practices
[^husain-synthetic]: Hamel Husain, Shreya Shankar. “What is the best approach for generating synthetic data?” Hamel’s Blog, 2025-06-01, modified 2025-07-27. https://hamel.dev/blog/posts/evals-faq/what-is-the-best-approach-for-generating-synthetic-data.html
[^aws-rag]: Lukas Wenzel, David Boldt, Johannes Langer. “Generate synthetic data for evaluating RAG systems using Amazon Bedrock.” AWS Machine Learning Blog, 2024-09-23. https://aws.amazon.com/blogs/machine-learning/generate-synthetic-data-for-evaluating-rag-systems-using-amazon-bedrock/
[^aws-ground-truth]: Samantha Stuart, Ivan Cui, Philippe Duplessis-Guindon, Rahul Jani. “Ground truth generation and review best practices for evaluating generative AI question answering with FMEval.” AWS Machine Learning Blog, 2025-03-05. https://aws.amazon.com/blogs/machine-learning/ground-truth-generation-and-review-best-practices-for-evaluating-generative-ai-question-answering-with-fmeval/
[^li-synthetic]: Zhuoyan Li, Hangxiao Zhu, Zhuoran Lu, Ming Yin. “Synthetic Data Generation with Large Language Models for Text Classification: Potential and Limitations.” EMNLP 2023. https://aclanthology.org/2023.emnlp-main.647/
