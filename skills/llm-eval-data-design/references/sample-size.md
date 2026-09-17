# 評価件数の決め方

## 先に決めること

普遍的な必要件数はない。次のどの目的で件数を決めるかを先に指定する。

1. 未知の失敗を見つける。
2. 成功率または失敗率を所定の精度で推定する。
3. 二つの候補の差を検出する。
4. 重大失敗率が所定の上限未満であることを確認する。
5. 人間とLLM採点器の一致を評価する。

## 初期の失敗発見

Anthropicは、製品要件、手動確認、現実の失敗から作る20〜50件の単純なタスクを開始点として挙げている。成熟したシステムや小さい差を測る場合は、より多く、より難しいタスクが必要になる。[^anthropic-evals]

20〜50件を、率の推定やリリース保証に必要な件数として使わない。初期集合では、件数より重要な層と期待する挙動を覆うことを優先する。

## 一つの比率を推定する

独立した二値結果を単純無作為抽出し、正規近似を使う場合は、次の式で概算する。[^nist-sample-size]

`n = z² × p × (1 − p) / e²`

- `n`: 必要件数
- `z`: 信頼水準に対応する値。95%信頼水準では約1.96
- `p`: 想定する比率。不明な場合は、必要件数が最大になる0.5を使う
- `e`: 信頼区間に許容する半幅

`p = 0.5`、95%信頼水準として切り上げると、次の概算になる。

| 許容する半幅 | 必要件数 |
| ---: | ---: |
| ±10パーセントポイント | 97件 |
| ±7パーセントポイント | 196件 |
| ±5パーセントポイント | 385件 |
| ±3パーセントポイント | 1,068件 |

この式は、全体の比率一つを推定する概算である。各層の比率に同じ精度が必要なら、層ごとに必要件数を確認する。小標本や成功・失敗が少ない場合は、Wilson区間または正確二項区間を使う。[^nist-binomial]

## 失敗が0件だった場合

独立な二項試行で0件の失敗を観測した場合、真の失敗率の片側95%上限は次で求める。

`上限 = 1 − 0.05^(1/n)`

`3/n`は近似値として使える。[^nist-binomial] 正確式から計算すると、0件の失敗を観測した場合に必要な件数は次になる。

| 片側95%上限の目標 | 必要件数 |
| ---: | ---: |
| 5%未満 | 59件 |
| 2%未満 | 149件 |
| 1%未満 | 299件 |
| 0.5%未満 | 598件 |
| 0.1%未満 | 2,995件 |

似たテンプレート、同じ文書、同じ会話から作った事例には相関があり得る。独立な試行という前提が疑わしい場合は、この表を保証値として使わない。

## 二つの候補を比較する

同じケースに二つのモデルまたは設定を適用する。結果には対応があるため、独立した二群として扱わない。二値判定ではMcNemar検定を候補にする。[^zhang-binary]

必要件数を決める前に、次を指定する。

- 有意水準
- 必要な検出力
- 検出したい最小差
- Aだけが成功する確率
- Bだけが成功する確率
- 一つのケースを繰り返す試行数

差、分散、不一致率が不明なら、予備評価から見積もる。検出力分析には仮定が必要であり、観測後の結果だけから必要件数を正当化しない。[^card-power]

## ケース数と試行数

- `task`: 異なる能力または利用状況を表すケース
- `trial`: 同じケースを同じ条件で繰り返した実行

10ケースを各10回実行しても、異なる100ケースにはならない。Anthropicは、非決定的なエージェント評価でtaskとtrialを区別している。[^anthropic-evals] 同じケース内の結果を独立に数えず、ケース単位の集計または階層的な分析を検討する。

## 報告する前提

- 推定または比較の目的
- 分母となるケースと除外条件
- 信頼水準または有意水準
- 許容誤差または最小検出差
- 各層の件数
- ケースごとの試行数
- 独立性を仮定できない要因
- 欠測、判定不能、採点不一致の扱い

[^anthropic-evals]: Mikaela Grace, Jeremy Hadfield, Rodrigo Olivares, Jiri De Jonghe. “Demystifying evals for AI agents.” Anthropic Engineering, 2026-01-09. https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
[^nist-sample-size]: NIST/SEMATECH. “Sample Sizes Required.” *e-Handbook of Statistical Methods*. https://www.itl.nist.gov/div898/handbook/prc/section2/old.prc272.htm
[^nist-binomial]: NIST. “Exact Binomial Confidence Limits.” Dataplot Reference Manual, 2010-10. https://itl.nist.gov/div898/software/dataplot/refman2/auxillar/exacbici.htm
[^zhang-binary]: Zhongheng Zhang et al. “Sample Size Calculations for Comparing Groups with Binary Outcomes.” *Shanghai Archives of Psychiatry*, 2017. https://pmc.ncbi.nlm.nih.gov/articles/PMC5738522/
[^card-power]: Dallas Card et al. “With Little Power Comes Great Responsibility.” EMNLP 2020. https://aclanthology.org/2020.emnlp-main.745/
