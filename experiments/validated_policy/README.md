# 拉面杯可选策略与配对验证

主属性接近上限时，原手写策略在“未满/已满”之间突然切换技能 PT 价格；超级拉面默认固定选第二项。本改动增加连续 PT 定价和按终盘属性缺口选择超级拉面范围的可选模式，默认值、游戏规则、正式 preset 和 game_config 均保持不变。

## 使用

已有 bench_base --tokens 和 RecommendedRamenTrainer::with_tokens 可启用：

- ptblend200-capd0-rgn1：固定卡组优先参考此组合。
- ptblend200-capd0-rgn1-supermode3：随机卡组可选；不保证每副卡组提升。
- ptblendN：连续窗口为 N/100 次当前训练的主属性增量；0 关闭，合法范围0～1000。窗口外、主属性已满、主属性增量为零均保持原行为。
- supermode0：默认固定选项二；1/2：固定一/三的实验对照；3：根据当前属性缺口及卡型数估值，平局优先原选项二。

capd0/rgn1 是已有开关，本 PR 没有修改它们的公式。supermode3 以每个训练位6×(50+30×卡数)估算增量，按现有真实属性上限和终局评分表计算价值，再乘(1+卡数)后按候选范围累加；它是启发式，不是六回合真实预测。

这些选项仅接入已有 Rust 构造/bench 入口，未增加线上配置直连字段，也未更改 MCTS rollout 默认策略。

## 独立验收结果

精确对照基线：04c739ca450804d834b71ffa1d011517a74c1b8a。以下取同一批第五轮独立验收，指标为完整77回合模拟的最终 calc_score，而非中间策略估值或 score_pt。不能把此前上游 GA 收益计入本次提升。

| 可选组合 | 固定卡组300局 | 7副预设2100局 | 160副随机卡组×300局 |
|---|---:|---:|---:|
| ptblend200-capd0-rgn1 | +263.64 | +185.96 | +85.92 |
| 再加supermode3 | +274.26 | +238.91 | +204.43 |

supermode3 相对前一组合的增量及近似95%区间：固定+10.62 [-34.78, +56.03]；预设+52.95 [+32.22, +73.69]；随机+118.51 [+84.47, +152.54]。随机160副中40副相对前一组合退步，最差约-267分。

冻结后跨4位马娘复核：随机64副、共38,400配对局，supermode3额外+151.84 [+94.65, +209.02]；固定600局为-21.73 [-55.76, +12.30]。固定组未证实额外收益，模拟结果不能直接当作线上实战或 MCTS 决策收益。

## 配对与抽样

每个候选与对照使用相同卡组、马娘、继承和 rule_master；异常保留、不补采。筛选与独立验收的普通卡身份分池，候选冻结后不再依据验收调参。

随机卡组从五种普通卡每类0～3张、合计5张的101种构成均匀采样；各类最新30张SSR按cardId排序奇偶分池、突破0～4均匀，固定满破友人303054，排除重复角色及育成角色。不代表全部玩家分布，不含SR/R及其他友人。random_diverse表示普通卡种类≥4，不等同规则中的deck_can_split。

随机置信区间按每卡组的配对均值聚类，跨马同卡组仍为一个聚类；固定和预设按配对局差分，均为正态近似95%区间。完整配置见冻结 JSON，重现时不要换用 bench_config 中可能不同的马娘/继承。

未纳入的实验也需说明：友人提前外出退步，弱位面板修正没有可靠收益，库存权重调整未通过验收，事件溢出修正没有改变实测选择，第六轮按后续隐藏用量统一成本使随机组退步。这些实现未加入本 PR。

## 复现

Rust 1.98.1、Release、锁定依赖；Python 3.11+，仅标准库。从仓库根执行：

~~~powershell
cargo build --release --locked -p umasim --no-default-features --bin policy_pair_bench
cargo test --release --locked -p umasim --no-default-features --lib pt_cap_blend
cargo test --release --locked -p umasim --no-default-features --lib test_round2
cargo test --release --locked -p umasim --no-default-features --lib super_choice
cargo test --release --locked -p umasim --no-default-features --lib test_yearly_observability
cargo test --release --locked -p umasim --no-default-features --bin policy_pair_bench
./target/release/policy_pair_bench.exe experiments/validated_policy/final-check.json target/policy-replay.csv
python experiments/validated_policy/analyze.py verify experiments/validated_policy/final-check.expected.csv target/policy-replay.csv
./target/release/policy_pair_bench.exe experiments/validated_policy/holdout.json target/policy-holdout.csv
python experiments/validated_policy/analyze.py analyze experiments/validated_policy/holdout.json target/policy-holdout.csv
./target/release/policy_pair_bench.exe experiments/validated_policy/cross.json target/policy-cross.csv
python experiments/validated_policy/analyze.py analyze experiments/validated_policy/cross.json target/policy-cross.csv
~~~

Linux/macOS去掉.exe。输出使用独立路径，以免覆盖冻结资料。默认配置含target-cpu=native，换机器需重新编译。

final-check.expected.csv保留8副×10种子×5臂共400条历史结果（仅去除耗时）；冻结JSON仅从历史清单去掉不再提供的evreal实验臂，卡组、种子和其他臂未改动。完整验收252,000条，跨马156,000条；分析器检查配对完整性、重复、异常及清单身份，并输出汇总JSON。原始大CSV、工具链和编译产物不纳入源码仓库，可由上述命令重新生成。
