# CoRL 2026 Rebuttal Issue Tracker - Submission 1544

目的：把 AC 和三位 reviewer 的所有问题拆成可跟踪条目，方便 rebuttal 前判断哪些问题已经通过实验或文字澄清覆盖。  
建议用法：每完成一个实验/分析，就把 `Status` 从 `TODO` 改成 `Done/Partial`，并在 `Evidence` 写入结果表格、图编号或 rebuttal 句子。

## 当前总体形势

Reviewer scores:  
Fdfz: 4 Borderline  
Ya8h: 3 Weak reject  
2Amw: 5 Weak accept  
AC: Proceed to rebuttal

AC 认为最核心的问题是：  
1. 缺少 external baselines  
2. Spatial anchor prediction 与 odometry 的关系和 novelty 没讲清  
3. Maze 环境不够真实，需要 realistic scanned layouts  
4. Drift / scale limit 没有分析  
5. Cross-attention 没有和简单融合方式比较  
6. Reward design / reward coefficient transfer 需要澄清  

## Rebuttal 优先级总表

| ID | Theme | Priority | Raised by | Issue | Best response | Evidence / result | Status |
|---|---|---:|---|---|---|---|---|
| E1 | External baseline | P0 | AC, Fdfz, Ya8h, 2Amw | 全文只有 GUIDE 内部消融，没有外部 baseline，无法证明优于已有方法 | 至少补 1-2 个外部 baseline。优先：SRU / spatial recurrent memory baseline；其次 Yang et al. [4] goal-initialized adaptation；再其次 REASAN [25] 或 VO/odometry-style baseline | 填：baseline名称、SR/AT/CR/FD、环境设置 | TODO |
| E2 | Proprioceptive odometry baseline | P0 | AC, Ya8h | Relative-spawn head 被认为就是 learned odometry；learning-based proprioceptive odometry 文献缺失 | 承认 relative-spawn head 可以视作 learned proprioceptive egomotion / odometry signal，但强调它不是 standalone localization module，而是 joint auxiliary representation；若可行，补 proprioceptive odometry + goal update baseline | 填：计算耗时、误差、导航结果 | TODO |
| E3 | Visual odometry / PointNav baseline | P1 | AC, Ya8h | AC 提到 Habitat no-GPS+Compass PointNav 和 Partsey et al. CVPR 2022；建议 visual-odometry-style baseline | 文字定位：GUIDE 与 no-GPS+Compass PointNav 目标类似，但 embodiment/sensing/control不同；如果有时间，补 VO-style baseline；否则 rebuttal 中承诺 camera-ready 补引用和讨论 | 填：是否补实验；若无，写引用和区别 | TODO |
| E4 | Realistic scanned layouts | P0/P1 | AC, Fdfz | 当前 DFS maze 无环路、单路径、统一走廊宽、直角路口，不像真实室内 | 尽量补 HM3D/Gibson floor plan projected occupancy terrain；如果时间不够，补更真实的 synthetic layouts：loops、variable width、irregular junctions、deep dead ends | 填：layout来源、数量、SR/FD、失败模式 | TODO |
| E5 | Deep dead-end / detour | P1 | Fdfz | Progress reward 假设朝目标近总是好；未测试需要暂时远离目标的非凸障碍/深死胡同 | 补 success vs dead-end depth；或者补 detour scenarios。解释 reward 并非唯一驱动，depth obstacle avoidance + exploration/curriculum 也约束行为 | 填：dead-end depth曲线或几个case结果 | TODO |
| E6 | Drift / scale limit | P0 | AC, 2Amw | 没分析 internal goal estimation error 如何随 episode length/path length/turn count 增长 | 补 drift curves：goal error vs path length, episode length, turn count；同时给 success degradation threshold | 填：曲线/表格，最大成功尺度 | TODO |
| E7 | Cross-attention ablation | P0/P1 | AC, Fdfz | 没证明 cross-attention 比 concat / late fusion 必要 | 补 cross-attention vs direct concat vs late fusion，同一训练设置或快速retrain；至少报告主要环境 SR/FD | 填：消融表 | TODO |
| E8 | Reward design clarity | P1 | AC, Fdfz, 2Amw, Ya8h | reward 公式和符号不清；line 164 某个参数没解释；GT variants 为什么不是100% | Rebuttal 中用 2-3 句说明 reward各项作用；解释 GT variant 仍受局部几何、低层控制、碰撞/FoV限制，不是oracle planner；camera-ready 会重写 reward section | 填：简化reward解释和GT failure原因 | TODO |
| E9 | Reward coefficient transfer | P1 | AC, Fdfz | 手调 reward coefficients 是否能迁移到不同 corridor width / obstacle density | 补或报告同一 coefficients 在新走廊宽度/障碍密度下结果；若无，承认并作为 camera-ready limitation | 填：transfer setting + SR/FD | TODO |
| E10 | Computational overhead | P1 | AC, Ya8h | 论文动机说外部定位有额外计算和延迟，但没实验支持；anchor predictor 是否比 VO/LiDAR odometry省 | 报告 GUIDE forward latency / anchor head overhead vs a representative odometry pipeline；强调没有 streaming localization dependency | 填：ms per step / Hz / device | TODO |
| E11 | Sensor robustness | P2 | Fdfz | 没测试 IMU drift、encoder errors、不同噪声profile、不同深度相机、遮挡/depth loss/motion blur | 如时间有限，不作为一页主打。可补一句：已有训练噪声覆盖 X；额外 robustness 留 camera-ready/future。若能补仿真噪声表更好 | 填：noise robustness结果 | TODO |
| E12 | Real-world quantitative details | P1 | AC, Fdfz | Fig.4 只有柱状图没有正文数字；真实环境复杂度和失败是否报告不充分 | 在 rebuttal 中给真实测试最大距离、maze复杂度、未报告失败数、SR/AT/CR/FD具体数值 | 填：真实实验数字 | TODO |
| E13 | Simulator missing | P2 | AC | 论文没有说明 simulator | Rebuttal/camera-ready 明确 simulator 名称和版本 | 填：simulator信息 | TODO |
| E14 | Error bars / confidence intervals | P2 | AC | 所有表/图缺 error bars / confidence intervals | 如有seed结果，补 mean±std；rebuttal一页可只写“we will add CI/error bars” | 填：mean±std | TODO |
| E15 | History buffer vs recurrent memory | P2 | Ya8h | history buffer 被称为 memory，但超过buffer长度的信息无效；错误分支时间超过buffer是否困住 | 若有实验，补 long wrong-branch/deep dead-end；否则澄清“memory”是短时 proprioceptive temporal context，不是 long-term map memory | 填：澄清句或实验 | TODO |
| E16 | Relative spawn and goal redundancy | P2 | Ya8h | 既预测相对起点又预测相对目标，是否冗余 | 解释二者提供互补监督：spawn head learns egomotion/path integration; goal head learns task-conditioned displacement; joint auxiliary improves representation | 填：若有 ablation更好 | TODO |
| E17 | Scale / higher-level planner | P1 | 2Amw | 环境尺度有限，大规模真实任务仍需要高层planner/localization，方法真实价值不清 | 承认 GUIDE 是 local/medium-range goal-initialized navigation module；强调它减少短程执行阶段 streaming goal updates，可作为 high-level planner 的 local executor | 填：范围界定句 | TODO |

## Reviewer-by-reviewer Comment Map

### Area Chair / Meta Review

| AC ID | Issue | Mapped tracker IDs | Notes |
|---|---|---|---|
| AC-1 | 没有 external baselines；Table 1 都是 GUIDE / ablation；Yang et al. 和 REASAN 只讨论没运行 | E1 | P0，必须回应 |
| AC-2 | Novelty questionable；spatial anchor relative-spawn head 是 odometry；缺 learning-based proprioceptive odometry 文献 | E2 | P0，建议承认一部分 |
| AC-3 | 缺 Habitat Challenge 2020 no-GPS+Compass PointNav / Zhao / Partsey CVPR 2022 讨论 | E3 | 主要是定位和引用 |
| AC-4 | DFS maze 结构不真实：无环、单路径、统一宽度、直角路口 | E4 | 最好补 realistic layouts |
| AC-5 | 建议 HM3D/Gibson floor plans 投影 occupancy 后作为 terrain | E4 | 若来不及，补更真实synthetic也比没有强 |
| AC-6 | 外部定位模块计算成本/延迟未测试 | E10 | Ya8h也提 |
| AC-7 | Scale limits 未测试 | E6, E17 | 和drift绑定回应 |
| AC-8 | Cross-attention vs concat / late fusion 未测试 | E7 | 相对容易补 |
| AC-9 | Drift analysis：goal-estimation error vs episode/path/turn count | E6 | P0 |
| AC-10 | Reward coefficient transfer to new corridor width / obstacle density | E9 | 可补小实验 |
| AC-11 | Clarify reward design | E8 | 文字回应 |
| AC-12 | Provide error bars / confidence intervals | E14 | minor |
| AC-13 | Quantitative real-world numbers absent in text | E12 | minor但好补 |
| AC-14 | Simulator never mentioned | E13 | minor但必须承认修 |

### Reviewer Fdfz - Score 4 Borderline

| Fdfz ID | Issue | Mapped tracker IDs | Notes |
|---|---|---|---|
| F-1 | Real-world tests are proof-of-concept in structurally similar/easy environments | E4, E12 | 需要真实复杂度数字或新增环境 |
| F-2 | No real difficult environments with deep dead-ends | E5 | 如果可补，能直接打中 |
| F-3 | Generalization claims overstated; no complexity metrics or unreported failures | E4, E12 | Rebuttal语气要收一点 |
| F-4 | Progress reward assumes moving toward goal is always beneficial | E5, E8 | 非凸绕障/暂时远离目标 |
| F-5 | Reward coefficients hand-tuned; no transfer evidence | E9 | 小实验或承认 |
| F-6 | Only internal ablations; no SLAM+planning or learning-based baselines | E1 | P0 |
| F-7 | Cross-attention not ablated against concat/late fusion | E7 | P0/P1 |
| F-8 | No robustness test to IMU drift, encoder errors, noise profiles | E11 | 如果没时间，弱回应 |
| F-9 | Ask max real navigation distance vs simulation | E12 | 数字即可 |
| F-10 | Ask whether real mazes as complex as simulation | E12 | 数字/描述 |
| F-11 | Ask whether unreported real-world failures exist | E12 | 诚实说明 |
| F-12 | Ask success vs dead-end depth | E5 | 实验最好 |
| F-13 | Ask detouring scenarios where reward misleading | E5 | 实验或解释 |
| F-14 | Ask depth camera transfer to Azure Kinect/ZED/ToF/stereo | E11 | 可作为 limitation |
| F-15 | Ask sensor failure: occlusion, depth loss, motion blur | E11 | 可作为 limitation |
| F-16 | Ask robustness to different proprioceptive sensor configs / sampling / drift | E11 | 可作为 limitation |
| F-17 | Ask why no external baselines | E1 | 不要解释“没时间”，直接补 |
| F-18 | Ask whether cross-attention ablated separately | E7 | 补表 |
| F-19 | Limitation section missing proof-of-concept/deep dead-end/reward assumption/baseline/sensor robustness | E5, E8, E11, E12 | camera-ready承诺 |

### Reviewer Ya8h - Score 3 Weak Reject

| Ya8h ID | Issue | Mapped tracker IDs | Notes |
|---|---|---|---|
| Y-1 | Motivation not convincing: localization overhead claim lacks experiments | E10 | P1 |
| Y-2 | Does spatial anchor prediction take less compute than visual/lidar odometry? | E10 | 同上 |
| Y-3 | Spatial anchor predicts relative spawn, i.e. odometry | E2 | P0 novelty风险 |
| Y-4 | Missing learning-based proprioceptive odometry literature | E2 | citation + baseline |
| Y-5 | Suggested proprioceptive odometry paper arXiv:2511.18857 | E2 | 需核对引用 |
| Y-6 | History buffer as memory; no effect beyond buffer length | E15 | 澄清/深死胡同实验 |
| Y-7 | Wrong branch longer than buffer may trap policy | E15, E5 | 和dead-end实验相关 |
| Y-8 | Relative spawn and relative goal predictions may be redundant | E16 | 文字解释/ablation |
| Y-9 | No external baseline; asks comparison from comment (2) and SRU | E1, E2 | P0 |
| Y-10 | Compare against "Spatially-enhanced recurrent memory for long-range mapless navigation via end-to-end reinforcement learning" | E1 | 用户提到SRU的话可放这里 |
| Y-11 | Line 164 symbol not explained clearly; maybe tunable constant | E8 | minor |
| Y-12 | Limitation says limited FoV failure, but reviewer thinks recurrent architecture should solve | E15, E11 | 可解释failure cause更复杂 |

### Reviewer 2Amw - Score 5 Weak Accept

| 2Amw ID | Issue | Mapped tracker IDs | Notes |
|---|---|---|---|
| A-1 | No baselines beyond own ablations | E1 | P0 |
| A-2 | Environment scale limited | E6, E17 | 要界定适用范围 |
| A-3 | Real-world tasks likely still need high-level planner/localization; value unclear | E17 | 文字定位 |
| A-4 | No analysis of internal target error vs episode/path/turn count | E6 | P0 |
| A-5 | Need drift curve | E6 | P0 |
| A-6 | Reward function design not clear | E8 | P1 |
| A-7 | Why ground-truth variants not 100% success? | E8 | P1 |

## Suggested One-page Rebuttal Structure

不建议逐 reviewer point-by-point，因为一页放不下。建议按 AC 关键问题聚合：

1. **Baselines and positioning.**  
   报告外部 baseline 结果；承认并定位 spatial anchor 与 learned odometry 的关系；补相关文献。

2. **Scalability and generalization.**  
   报告 drift curves / scale limits；如果有 realistic layouts 或 deep dead-end 实验，在这里放结果。

3. **Architecture and reward clarifications.**  
   报告 cross-attention vs concat/late fusion；简述 reward 设计、reward coefficient transfer、GT variant not 100% 原因。

4. **Minor corrections.**  
   一句话列出 camera-ready 会补：simulator、real-world numbers、error bars、related work、limitation wording。

## Minimal Rebuttal Success Criteria

一页 rebuttal 最好至少包含这些新结果：

| Must-have | Why |
|---|---|
| 外部 baseline 表 | 三位 reviewer + AC 都提，是最大硬伤 |
| Drift / scale curve | AC 和 WA reviewer 明确要求，能证明你知道方法边界 |
| Cross-attention ablation | 相对容易补，能显示 architecture choice 不是随便加 |
| Odometry positioning paragraph | 保护 novelty，避免被认为“就是odometry换名字” |

如果只能补两个实验，优先顺序：
1. External baseline
2. Drift / scale analysis
3. Cross-attention ablation
4. Realistic / deep-dead-end layouts

## 原文逐条对照

这一节保留 reviewer / AC 的原始英文表述，方便核对上面的中文归纳是否准确。  
写 rebuttal 时建议优先引用 AC 的 `Key Issues for Rebuttal`，因为最终 AC/SAC 会按这些核心问题综合判断。

### Area Chair / Meta Review - Original Text

#### AC Summary / Pros / Cons

**Summary**

> The authors propose GUIDE, an end-to-end method for goal-initialized navigation where the robot must navigate without streaming goal or pose updates. The policy fuses depth observations with multi-frequency proprioception streams via self-attention and cross-attention, and outputs velocity commands tracked by a low-level locomotion controller. An auxiliary spatial anchor predictor is supervised against privileged states to estimate body velocity, relative target, and relative spawn position. The authors train using asymmetric PPO with a privileged map-based critic and a curriculum over cluttered and maze terrain families. They report 99.6-99.9% SR across four benchmark settings, and demonstrate zero-shot sim2real transfer in an 8×8 m lab environment.

**Pros**

> The method removes the need for localization to provide streaming goal updates, which Reviewer 2Amw notes makes the assumption more suitable for real-world deployment. All three reviewers found the method description clear and the ablations comprehensive in demonstrating the contribution of each component. Zero-shot sim2real experiments show the policy transfers to hardware.

**Cons: external baselines**  
Mapped: `E1`

> All three reviewers flagged that the authors do not compare against any external baselines. Every row of Table 1 is GUIDE or a GUIDE ablation, and Yang et al. [4] and REASAN [25] are discussed in Section 2.2 but never run.

**Cons: novelty / odometry / missing literature / realistic layouts**  
Mapped: `E2`, `E3`, `E4`

> Second, the novelty is questionable and the citations are inadequate. Reviewer Ya8h observes that the spatial anchor predictor's relative-spawn head is itself odometry, and that the learning-based proprioceptive odometry literature is entirely absent. In addition, the Habitat Challenge 2020 Realistic PointNav track removed GPS+Compass, spawning an established line of work: Zhao et al. (71.7% success in Habitat simulation) and Partsey et al., CVPR 2022 [1] (94% success in Habitat simulation, with a separate zero-shot sim2real demonstration on a LoCoBot). Additionally, the maze terrains are generated via randomized DFS, which yields loop-free layouts with exactly one valid route per start–goal pair, uniform corridor widths, and right-angle junctions only, which is structurally unlike real interiors. Please evaluate on layouts derived from scanned building datasets (e.g. HM3D or Gibson floor plans projected to occupancy and used as terrain), which would test whether the learned spatial context holds where loops, irregular geometry, and varied corridor scale exist. Note that the no-GPS+Compass PointNav literature cited above evaluates exactly these scanned environments, which would also make the comparison more direct.

**Cons: other missing tests**  
Mapped: `E7`, `E10`, `E6`

> Several other points are not tested in the paper, such as the additional computational cost and latency of an external localization module (Reviewer Ya8h), scale limits (Reviewer 2Amw), and cross-attention fusion versus concatenation or late fusion (Reviewer Fdfz).

#### AC Key Issues for Rebuttal - Original Text

**External baselines**  
Mapped: `E1`, `E2`, `E3`

> External baselines (all three reviewers). Please compare against an external baseline (for an example, Yang et al. [4] adapted to the goal-initialized setting). Reviewer Ya8h additionally requests a learning-based proprioceptive-odometry comparison, and a visual-odometry-style baseline in the spirit of Partsey et al. would also be informative.

**Realistic environments**  
Mapped: `E4`

> Evaluate against realistic environments. Please evaluate on layouts derived from scanned building datasets (e.g. HM3D or Gibson floor plans projected to occupancy and used as terrain), which would test whether the learned spatial context holds where loops, irregular geometry, and varied corridor scale exist. This also allows difficulty calibration with prior PointNav literature.

**Cross-attention ablation**  
Mapped: `E7`

> Cross-attention ablation (Reviewer Fdfz). Compare against direct concatenation or late fusion to justify the added complexity.

**Drift analysis**  
Mapped: `E6`

> Drift analysis (Reviewer 2Amw). Please report how internal goal-estimation error scales with episode length, path length, and turn count, and identify the scale at which success degrades. This matters because the anchor heads are supervised on the proprioceptive token alone and receive no visual correction, so drift accumulates without bound.

**Position against navigation without streaming pose updates**  
Mapped: `E3`

> Position against navigation without streaming pose updates. Please cite and discuss the Habitat Challenge 2020 Realistic PointNav track, with substantial follow-on work (Zhao et al.; Partsey et al., CVPR 2022, reporting 94% success in Habitat simulation with a separate zero-shot sim2real demonstration on a LoCoBot).

**Spatial anchor prediction vs learned odometry**  
Mapped: `E2`, `E10`

> Address whether spatial anchor prediction is learned odometry (Reviewer Ya8h). If it differs, say how; if not, the novelty claim, related work, and baselines should engage that literature. Please also report the computational comparison Ya8h requested regarding the motivation claim about localization overhead in L42-45 .

**Reward-coefficient transfer**  
Mapped: `E9`

> Reward-coefficient transfer (Reviewer Fdfz). Do the hand-tuned coefficients transfer to new corridor widths and obstacle densities without retuning?

**Reward design**  
Mapped: `E8`

> Clarify the reward design (Reviewers Ya8h and 2Amw).

**Minor points**  
Mapped: `E12`, `E13`, `E14`

> Provide error bars / confidence intervals for all tables / figures.
>
> Quantitative real-world results. Fig. 4 reports SR, AT, CR, and FD as bar charts with no numbers in text.
>
> The simulator is never mentioned in the paper.

### Reviewer Fdfz - Original Text

#### Fdfz Summary / Strengths

**Summary**

> This paper proposes GUIDE, an end-to-end reinforcement learning framework for legged robot navigation that operates without continuous external localization. GUIDE works from a single initial target specification. The robot must then navigate using only its internal spatial memory, without subsequent external guidance. The framework cultivates internal directional awareness through a spatial anchor predictor that processes multi-frequency proprioceptive data to extract egomotion representations. This enables the robot to maintain persistent long-horizon spatial context while using raw depth streams to perceive local environmental geometry. Experimental evaluation on a quadruped robot across both simulation and real-world scenarios demonstrates that GUIDE is able to learn reliable egomotion and directional awareness. The fully end-to-end deployed policy navigates safely through dense clutter and structured mazes without requiring subsequent goal guidance or prior maps.

**Strength: initial goal only**  

> The method requires only an initial relative target position in the robot's egocentric frame, with no streaming target updates during rollout. This is practically convenient as it removes the need for continuous external localization modules.

**Strength: spatial anchor + depth**  

> The spatial anchor predictor effectively cultivates internal egomotion and directional awareness by supervising the estimation of body velocity, relative target position, and relative spawn position. Combined with depth image processing, the method generalizes well across visual variations since no RGB is used—depth is robust to lighting changes.

**Strength: high success / real final distance**  

> The method achieves 99.9% success rate in cluttered environments and 99.6% in hard maze environments in simulation, with real-world final distance to goal of only 0.2–0.3 meters, demonstrating high precision in spatial estimation.

#### Fdfz Weaknesses / Questions / Limitations

**F-1 / F-2 / F-3: real-world proof-of-concept, deep dead-ends, overstated generalization**  
Mapped: `E4`, `E5`, `E12`

> The in-the-wild tests are proof-of-concept in structurally similar environments. No evaluation of real difficult environments with "deep dead-ends" where the progress reward would fail. Generalization claims are overstated. No metrics on environment complexity or unreported failures. Best characterized as demonstrations rather than rigorous validation.

**F-4: progress reward assumption**  
Mapped: `E5`, `E8`

> Progress reward assumes moving toward goal is always beneficial. No testing of scenarios requiring temporary movement away from goal (detouring around obstacles), where reward would penalize correct behavior.

**F-5: reward coefficient transfer**  
Mapped: `E9`

> Reward coefficients appear hand-tuned with no evidence they transfer across environments without retuning. No discussion of whether manual tuning is needed for new deployment scenarios.

**F-6 / F-17: no external baselines**  
Mapped: `E1`

> Only internal ablations performed. No comparison with SLAM+planning pipelines or other learning-based methods. Cannot claim superiority without external baselines.

**F-7 / F-18: cross-attention ablation**  
Mapped: `E7`

> The paper uses cross-attention to fuse proprioceptive and visual features, but never tests whether this specific mechanism is actually necessary, such as by comparing it against simpler alternatives like direct feature concatenation or late fusion, leaving it unverified whether the added computational complexity is justified.

**F-8 / F-14 / F-15 / F-16: sensor robustness**  
Mapped: `E11`

> No quantitative testing of robustness to IMU drift, encoder errors, or different noise profiles. Robustness to sensor imperfections is unverified.

**F-9 / F-10 / F-11: real-world complexity gap and unreported failures**  
Mapped: `E12`

> The real-world tests are done in relatively easy environments. Could you clarify the complexity gap between simulation and real-world tests? Specifically:
> What was the maximum navigation distance in real-world vs. simulation?
> Were the real-world mazes/maze-like structures as complex as the simulation mazes?
> Were there any real-world scenarios where the method failed that weren't reported?

**F-12 / F-13: dead-end depth and misleading progress reward**  
Mapped: `E5`

> The paper tests maze-like terrains with dead ends. However, we note that scenarios with "deep dead-ends" (where the goal is nearby but the path out is long) are not explicitly tested. The progress reward r_prog = max(0, d_best - d_t) / d_t=0 assumes progress toward the goal is always good. In environments with non-convex obstacles (e.g., requiring temporary movement away from the goal to find a path), does this reward shaping cause local minima? Have you evaluated:
> Success rate as a function of dead-end depth (number of decisions to escape)?
> Scenarios where the progress reward would be misleading (e.g., detouring around obstacles)?

**F-5 expanded: coefficient transfer**  
Mapped: `E9`

> The reward function includes multiple hand-tuned coefficients (σ_pos = 0.5, σ_head = 0.5, etc.) and coefficients. Is there evidence that these values transfer without retuning to new environments? If the method were deployed to a new building with different corridor widths or obstacle densities, would the same coefficients work, or would they require manual tuning?

**F-14: depth camera model transfer**  
Mapped: `E11`

> The experiments use Intel RealSense D435i with specific simulation noise models. Have you tested with different depth camera models (e.g., Azure Kinect, ZED, time-of-flight vs. stereo)? If the method were deployed to a robot with a different sensor, would it require retraining?

**F-15: camera failure / depth quality**  
Mapped: `E11`

> The real-world tests mention "limited field-of-view" as a challenge. Have you tested sensor failure scenarios (e.g., temporary camera occlusion, complete depth loss, motion blur during fast locomotion)? How does the method degrade when depth quality drops below training noise levels?

**F-16: proprioceptive sensor configuration / drift**  
Mapped: `E11`

> The ablation study shows high-frequency proprioception (200Hz) is crucial. However, different robot platforms may have different IMU/encoder specifications (sampling rates, noise profiles, drift characteristics). Is the method robust to different proprioceptive sensor configurations so no retraining or fine-tuning required? Also, the paper doesn't explicitly test robustness to IMU drift or encoder errors. Have you tested how the method works with (simulated or real) situations where such drift and/or error exist?

**F-17: why no external baselines**  
Mapped: `E1`

> The paper does not compare with external baseline methods (only internal ablations). Could you clarify why comparisons with existing approaches were omitted?

**F-18: cross-attention separately ablated?**  
Mapped: `E7`

> Was the cross-attention mechanism between proprioception and vision ablated separately?

**F-19: missing limitations**  
Mapped: `E5`, `E8`, `E11`, `E12`

> The paper's Limitation section does not discuss: (i) proof-of-concept real-world tests lacking rigorous validation or deep dead-ends, (ii) reward design assuming progress toward goal is always beneficial, with no testing of detouring scenarios, (iii) absence of baseline comparisons, (iv) unverified robustness to IMU drift, encoder errors, or different sensors.

**Fdfz score / confidence**

> Overall Score: 4: Borderline. The paper has interesting ideas but notable weaknesses — e.g., insufficient experiments, unclear contribution, or limited novelty. It is unlikely authors could address all the issues in the limited period of the rebuttal.
>
> Confidence Score: 4: High confidence. I am knowledgeable in this area and confident in my assessment.

### Reviewer Ya8h - Original Text

#### Ya8h Summary / Strengths

**Summary**

> This paper presents an end-to-end navigation policy for legged robots. The major novelty is using a network to predict spatial anchors from proprioception data so that no localization is needed during navigation. Simulation and real-world experiments show that the policy can navigate through challenging environments like Maze.

**Strengths**

> The description of the method is clear. The ablation study is comprehensive, and the real-world experiments are appreciated.

#### Ya8h Weaknesses / Questions / Limitations

**Y-1 / Y-2: motivation and computational overhead**  
Mapped: `E10`

> The motivation does not seem convincing enough. The author claims "This necessitates an additional localization module to frequently update the goal and robot positions, incurring additional computational costs and latency" in the introduction. This is expected to be supported by experimental results. Does the spatial anchor prediction take less computational power than a normal visual/lidar odometry?

**Y-3 / Y-4 / Y-5: spatial anchor as odometry and missing literature**  
Mapped: `E2`

> Navigation without explicit odometry is one of the main novelties in this paper. However, the spatial anchor prediction still explicitly predicts the relative position to spawn, which is in other words an odometry. In this case, the literature study about learning-based proprioception odometry is highly relevant but not mentioned at all (e.g. [https://arxiv.org/pdf/2511.18857](https://arxiv.org/pdf/2511.18857)). These can also be strong baselines to compare against in the experiments.

**Y-6 / Y-7: history buffer as memory**  
Mapped: `E15`, `E5`

> In the methodology part, the history buffer is introduced as memory. Compared with a recurrent network, the memory before the buffer has no effect on the policy at all. I wonder if there is a wrong branch in the maze in which the robot spends more time than the buffer size, will it be trapped in the local region?

**Y-8: relative spawn / goal redundancy**  
Mapped: `E16`

> In spatial anchor prediction, relative positions to both the spawn and goal are predicted, but are they redundant since it is assumed the robot knows the relative position between spawn and goal?

**Y-9 / Y-10: no external baseline and requested comparisons**  
Mapped: `E1`, `E2`

> The experiment is solid if seen as only an ablation study, but there is no other baseline implemented. It is appreciated that the comparison mentioned in comment (2) and the comparison against "Spatially-enhanced recurrent memory for long-range mapless navigation via end-to-end reinforcement learning" are shown.

**Y-11: unexplained symbol / parameter**  
Mapped: `E8`

> In line 164, 
>  is not explained clearly. I guess it's a constant parameter to be tuned?

**Ya8h questions**

Mapped: `E10`, `E2`, `E1`

> It would be appreciated to add a computational time comparison between Spatial Anchor Prediction and normal odometry to support the motivation.
>
> If the Spatial Anchor Prediction is different from learning-based odometry, the author needs to make it clear. Otherwise, the novelty claims, related works, and baseline comparison should all include relevant works.
>
> It would be appreciated to add a comparison with "Spatially-enhanced recurrent memory for long-range mapless navigation via end-to-end reinforcement learning".

**Y-12: limited FoV limitation disputed**
Mapped: `E15`, `E11`

> The limitation of limited FoV is discussed. However, I don't think it is the reason for failure in the shown case. The recurrent architecture should be able to solve this.

**Ya8h score / confidence**

> Overall Score: 3: Weak reject. Below the acceptance threshold. The paper has identifiable merit but significant weaknesses — e.g., missing key comparisons, unconvincing results, or incremental contribution.
>
> Confidence Score: 4: High confidence. I am knowledgeable in this area and confident in my assessment.

### Reviewer 2Amw - Original Text

#### 2Amw Summary / Strengths

**Summary**

> The paper tackles goal-initialized visual navigation for a quadruped: the relative goal is provided only once at episode start, and the policy must navigate without any streaming goal/pose updates from an external localization module. The core method, GUIDE, is an end-to-end PPO policy that fuses raw depth with multi-frequency proprioception, and trains an auxiliary spatial anchor predictor with three heads (body velocity, relative goal position, relative spawn position). The proposed method is evaluated in cluttered + maze terrains in simulation and via zero-shot real-world deployment.

**Strengths**

> This paper removes streaming goal updates during mapless navigation, making the assumption more suitable for real-world deployment.
>
> The network design is overall sound, and the ablation demonstrates the effectiveness of core designs.
>
> Zero-shot sim-to-real experiments further validate the proposed method.
>
> The authors provide interesting insights on high-frequency proprioceptive data for state estimation learning.

#### 2Amw Weaknesses / Questions / Limitations

**A-1: no external baselines**  
Mapped: `E1`

> No baselines beyond ablations of the authors' own method. Table 1 compares only GUIDE variants. There is no comparison against a representative prior method. Without this, it is hard to judge whether GUIDE's design is better than existing methods.

**A-2 / A-3: environment scale and real-world value**  
Mapped: `E6`, `E17`

> The scale of the environment that the proposed method looks limited. To deploy this method in real-world tasks, higher-level planners seem still necessary (so does the localization), making the value of the proposed method in real-world tasks unclear.

**A-4 / A-5: internal target error / drift**  
Mapped: `E6`

> There is no analysis of how internal target error grows with episode length/path length/turn count. The proposed method is not likely to be able to work in a large-scale environment. It is important to know its upper bound for successful completion.

**2Amw questions**
Mapped: `E6`, `E8`

> How does internal goal-estimation error scale with episode/path length and number of turns? Can you show a drift curve?
>
> The Reward Function Design is not well explained, and I found it not easy to follow. Please try to make it clearer.
>
> Why can't the ground truth variants achieve 100% success rates? Considering the scale of the environment tackled in this work, this task should be easy if the ground truth target position is accessible.

**2Amw limitations / broader impact**

> The limitation section is concrete and honest. It names a specific failure mode and backs it with a dedicated figure.

**2Amw score / confidence**

> Overall Score: 5: Weak accept. A good paper with merit that outweighs its weaknesses. The contribution is valid but may have gaps in evaluation, limited novelty over prior work, or clarity issues that the authors could address.
>
> Confidence Score: 4: High confidence. I am knowledgeable in this area and confident in my assessment.
