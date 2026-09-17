# 層別化と配分

## 層別化の目的

層別化は、評価対象を意味のある区分に分け、重要な区分の見落としを防ぐために行う。CheckListは、言語能力とテスト種別の行列によってテスト発想を整理する。[^checklist] HELMは、利用場面と評価指標から評価空間を分類する。[^helm]

## 最初に定義する三軸

### 業務フローまたは利用目的

利用者が完了したい仕事に基づいて分ける。画面、API、プロンプトの形ではなく、検索、比較、要約、申請、予約、判断支援などの目的で定義する。

### 失敗時の重大度

少なくとも低、中、高、重大に分ける。名称を変える場合は、各段階の判定条件を記述する。NIST AI RMFは、影響、発生可能性、利用可能な資源に基づいてリスク対応の優先順位を決めるよう求めている。[^nist-ai-rmf]

### 期待する挙動

次のいずれかを明示する。

- 実行する
- 拒否する
- 追加情報を確認する
- 判断を保留する
- 人間または別の処理へ引き継ぐ

肯定例だけでなく、同じ能力を使ってはいけない事例を含める。

## 必要に応じて追加する軸

- 難易度: 容易、中間、困難、現状では解答不能
- 入力: 長さ、形式、誤字、曖昧さ、情報不足、複数意図
- 利用者: 役割、知識量、目的、感情状態、言語
- 文脈: 単発、複数ターン、長い履歴、雑音、矛盾
- システム経路: 検索、ツール、権限、外部依存先、引き継ぎ
- 安全性: 通常利用、有害依頼、直接または間接のプロンプト注入、権限逸脱
- データの由来: 専門家、仕様、業務文書、合成、手動試験、本番、インシデント

## 組み合わせ爆発を抑える

全軸の直積を作らない。次の順序で優先する。

1. 重大度が高い組み合わせを含める。
2. 中核フローごとに、実行、拒否、確認のうち該当する挙動を含める。
3. 既知の失敗仮説と境界条件を含める。
4. 二つの軸の組み合わせを広く覆う。
5. 無効な組み合わせを除外する。
6. 生成後に意味的重複を確認する。

## 各層への配分

| 目的 | 配分方法 | 報告時の注意 |
| --- | --- | --- |
| 本番全体の率を推定する | 本番または想定する層比率に比例させる | 本番前の比率は仮説として示す |
| 層同士を比較する | 小さい層にも最低件数を確保する | 単純平均を全体率にしない |
| 重大な失敗を探す | 高リスク層を過剰抽出する | 層別結果として報告する |
| 費用を抑えて精度を高める | 層の分散と費用を使って配分する | 推定に使った値と前提を示す |

Statistics Canadaは、等配分、比例配分、Neyman配分、費用最適配分を区別している。[^statistics-canada-allocation] Neyman配分は、層の規模と標準偏差に応じて標本を配分する。必要な分散が不明な段階では、予備評価で見積もる。

## 重み付け

過剰抽出した標本から本番全体の率を推定する場合は、層の母集団比率または抽出確率に基づく重みを使う。[^census-weighting] ただし、合成データだけでは未知の本番比率を復元できない。リリース前の配分を、カバレッジ設計として明記する。

## レビュー項目

- 各層が、製品上の意味を持っているか。
- 同じ概念を別名の層として重複させていないか。
- 高リスク層が、件数の少なさだけで除外されていないか。
- 実行すべき事例と、実行すべきでない事例が含まれているか。
- すべての組み合わせを機械的に作っていないか。
- 本番比率、設計上の配分、実際に得られた件数を区別しているか。
- 層別値を報告する層に、必要な有効件数があるか。

[^checklist]: Marco Tulio Ribeiro, Tongshuang Wu, Carlos Guestrin, Sameer Singh. “Beyond Accuracy: Behavioral Testing of NLP Models with CheckList.” ACL 2020. https://aclanthology.org/2020.acl-main.442/
[^helm]: Percy Liang et al. “Holistic Evaluation of Language Models.” *Transactions on Machine Learning Research*, 2023. https://openreview.net/forum?id=iO4LZibEqW
[^nist-ai-rmf]: National Institute of Standards and Technology. “Artificial Intelligence Risk Management Framework (AI RMF 1.0).” NIST AI 100-1, 2023-01. https://doi.org/10.6028/NIST.AI.100-1
[^statistics-canada-allocation]: Statistics Canada. “Section 4. Sample size determination.” *Survey Methodology*, 2020. https://www150.statcan.gc.ca/n1/pub/12-001-x/2020002/article/00001/04-eng.htm
[^census-weighting]: U.S. Census Bureau. “Weighting.” 2022-08-18. https://www.census.gov/programs-surveys/sipp/methodology/weighting.html
