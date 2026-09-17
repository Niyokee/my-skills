# 評価集合の種類と用途

## 基本原則

一つの評価集合に、失敗発見、回帰検知、本番率推定という異なる目的を持たせない。目的が違えば、必要な事例の分布と合格条件も違う。

OpenAIは、低頻度で重大なリスクを覆う標的型評価と、実運用に近い頻度を推定する配置シミュレーションを区別している。[^openai-deployment] Anthropicは、新しい能力の限界を測る評価と、既存能力の後退を検出する評価を区別している。[^anthropic-evals]

## 能力・リスク評価集合

未知の弱点、境界条件、重大事故を見つけるために使う。

- 本番で稀でも、影響が大きい事例を厚くする。
- 典型例、境界例、敵対的事例を含める。[^openai-best-practices]
- 実行すべき事例だけでなく、拒否、確認、保留、引き継ぎが必要な事例を含める。
- 全体平均より、層別の失敗内容と重大度を報告する。
- 過剰抽出した重大事例の割合を、本番での発生頻度として扱わない。

## 回帰評価集合

一度確認した失敗の再発と、中核機能の後退を検出するために使う。

- 本番、手動試験、能力・リスク評価で確認した失敗を固定する。
- 修正が完了した時点の入力、期待結果、実行条件を保存する。
- 製品の中核フローについて、少数の正常事例も固定する。
- ケースの追加理由と、関連する失敗分類を記録する。
- 現在の本番分布を推定する集合として扱わない。

## 代表性評価集合

対象母集団における成功率、失敗率、費用、遅延などを推定するために使う。

- 本番後は、定義した期間の本番ログから確率的に抽出する。
- 層別抽出する場合は、抽出確率と集計用の重みを記録する。
- 本番前は、想定した利用分布であることを明記する。
- 想定分布と本番分布の差を、リリース後に確認する。
- 信頼区間と、推定対象となる期間、利用者、機能を報告する。

比例配分と不比例配分では、標本から得られる層別精度と全体精度が変わる。[^statistics-canada] 選択確率が異なる標本から母集団の値を推定する場合は、重みを使って不均等な代表性を補正する。[^census-weighting]

## 開発用集合と最終判定用集合

- 開発用集合は、プロンプト、検索、ツール、採点基準を改善するために繰り返し使う。
- 最終判定用集合は、リリース判断まで内容を必要以上に公開しない。
- 同じ失敗分類を両方に含めてもよいが、同一ケースの重複は記録する。
- 最終判定用集合を開発に使った場合は、そのケースを開発用へ移し、新しい保留ケースを補充する。

## 集計規則

- 三つの評価集合の単純平均を総合成功率として報告しない。
- 能力・リスク評価集合は、層別結果と重大な失敗事例を報告する。
- 回帰評価集合は、ケース別合否と前版からの変化を報告する。
- 代表性評価集合は、必要に応じて重み付けした推定値と信頼区間を報告する。
- 複数の評価集合に属するケースは、各集合の分母を明示する。

[^openai-deployment]: OpenAI. “Predicting model behavior before release by simulating deployment.” 2026-06-16. https://openai.com/index/deployment-simulation/
[^anthropic-evals]: Mikaela Grace, Jeremy Hadfield, Rodrigo Olivares, Jiri De Jonghe. “Demystifying evals for AI agents.” Anthropic Engineering, 2026-01-09. https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
[^openai-best-practices]: OpenAI. “Evaluation best practices.” OpenAI Developers, 2026-09-17閲覧. https://developers.openai.com/api/docs/guides/evaluation-best-practices
[^statistics-canada]: Statistics Canada. “Section 4. Sample size determination.” *Survey Methodology*, 2020. https://www150.statcan.gc.ca/n1/pub/12-001-x/2020002/article/00001/04-eng.htm
[^census-weighting]: U.S. Census Bureau. “Weighting.” 2022-08-18. https://www.census.gov/programs-surveys/sipp/methodology/weighting.html
