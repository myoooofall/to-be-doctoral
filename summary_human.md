# H2O
reference:Learning Human-to-Humanoid Real-Time Whole-Body Teleoperation
![H2O_pipeline ](./humanoid/H2O_pipeline.png)  
感觉还是AMP那套

# maskmimic
说白了就是先用一个teacher训一个给全部命令和约束的  然后蒸馏一个输入部分命令或约束的 student policy 起到一个最终只有一点约束或命令 就能生成全部动作的效果  

# BFM-Zero: A Promptable Behavioral Foundation Model for Humanoid Control Using Unsupervised Reinforcement Learning

- **时间/发表**：2025-11 首次发布；正式发表于 **ICLR 2026**。
- **作者/单位**：Yitang Li、Zhengyi Luo 等；CMU LeCAR Lab、Meta 等。
- **机器人与仿真**：主要为 29-DoF Unitree G1；Isaac Lab 训练、MuJoCo sim-to-sim 检验，并在真实 G1 上部署；附录还展示了迁移到 Booster T1 的结果。
- **算法定位**：Forward-Backward representation + FB-CPR，属于**在线采集数据、off-policy、无任务奖励预训练的 actor-critic**，不是 PPO。
- **原文与代码**：[论文](https://arxiv.org/abs/2511.04131) · [ICLR 2026 Proceedings](https://proceedings.iclr.cc/paper_files/paper/2026/hash/80b677bb4d17f77220be88e1d1fdebde-Abstract-Conference.html) · [项目页](https://lecar-lab.github.io/BFM-Zero/) · [官方代码](https://github.com/LeCAR-Lab/BFM-Zero)。

## 一句话总结

BFM-Zero 不像常见人形工作那样为每段 reference 用 PPO 训练一个 motion tracker，也不先把 tracker 蒸馏成 latent action decoder。它在仿真中让一个由 latent `z` 条件化的策略族自由交互，并用 Forward-Backward 表征学习「每个 `z` 会使机器人长期访问哪些状态」。训练结束后，目标姿态、参考动作或新 reward 都可通过统一公式转换成 `z`，再交给同一个策略执行，从而实现零样本 goal reaching、motion tracking 和 reward optimization。

## BFM 到底是什么

**BFM = Behavioral Foundation Model，行为基础模型。**它不是一个像 Transformer、PPO 那样定义明确的网络结构，也不是某个软件库，而是一种系统定位：先训练一个覆盖大量机器人行为的通用控制模型，之后用不同 prompt 调用同一个模型完成不同任务，或者用少量交互快速适配。

作者认为，一个理想 BFM 应具备：

- 一个共享策略，而非每项技能重新训练一套策略；
- 一个统一的 task/behavior prompt 接口；
- 对新目标能够 zero-shot 执行，困难任务也能快速 post-training；
- 在真实机器人上具有足够的稳定性。

这里的 BFM 具体由 **latent-conditioned policy `π(o_history,z)`、Forward map `F`、Backward map `B`、风格/安全 critics** 组成。“Zero”指训练完成后可以不再优化网络，直接由目标、轨迹或 reward 算出 prompt `z`。它不是语言基础模型，也没有视觉/语言输入。

还要注意，2025 年另有一篇标题就叫 **Behavior Foundation Model for Humanoid Robots**（arXiv:2509.13780）。BFM 同时被用作越来越宽泛的领域名称；本文的具体方法叫 **BFM-Zero**，底层是 Forward-Backward 无监督 RL，不应把所有名为 BFM 的工作视为同一网络。

## 它为什么不用 PPO

你的判断是对的：**这篇没有使用 PPO。**二者差异不是只换了一个 loss：

| PPO 式人形控制 | BFM-Zero |
|---|---|
| on-policy；当前策略采完的数据更新几轮后通常丢弃 | off-policy；交互数据存入约 500 万 transition 的 replay buffer，反复采样更新 |
| 用 advantage 和 clipped policy objective 更新策略 | 用 TD/Bellman loss 学 `F、B、Q_D、Q_R`，actor 直接最大化这些 critics 的评价 |
| 通常预先给定 tracking/locomotion task reward | 预训练时不给下游 tracking、goal、velocity 等任务 reward，随机采样 `z` 学整个策略族 |
| 一套 policy 通常对应给定命令族或 reference task | 一个 `π(o,z)` 通过不同 `z` 表示不同目标/任务 |

从工程结构看，它更接近 **DDPG/TD3 一类 replay-buffer + TD critic + actor 直接最大化 critic 的 off-policy actor-critic**，但也不能简单叫 TD3：它没有 TD3 的标准双 Q 目标，而是使用 Forward-Backward successor representation，并额外训练动作风格和安全 critics。最准确的名称是 **FB-CPR 基础上的 off-policy actor-critic**。

## 输入和输出

真实部署时 actor 输入：

- 当前及过去 4 步本体状态和历史动作；
- 当前关节位置、关节速度；
- root angular velocity 与 projected gravity；
- 256 维 task/behavior prompt `z`。

actor **不接收**仿真中的全身 link pose、全局线速度等 privileged state。训练时 `F、B、Q_D、Q_R` critics 可以读取 463 维完整状态，形成 asymmetric actor-critic；部署 actor 只靠可测本体历史。

输出为 29 维全身 **PD position targets**，再由底层 PD 产生关节力矩。它不是 torque policy，也没有视觉输入。仿真运行 200 Hz，控制策略运行 50 Hz。完整训练系统约 440.5M 参数，其中 actor 约 31.9M；部署主要运行 actor 和外部给定/预计算的 `z`，不是把所有 critics 都装到真机上。

## Forward-Backward 到底在学什么

这是全文最难理解的部分。先不要看公式，可以把它分成四个量：

### `B(s')`：给未来状态建立坐标

Backward map `B` 把某个可能到达的状态 `s'` 编成一个 256 维向量。这个向量不是一帧动作，也不是 reference 压缩器；它是在学习「这个状态在长期行为结构中代表什么」。名字叫 backward，不是倒放动力学，而是后续从目标状态反推出 task prompt 时使用这个映射。

### `F(s,a,z)`：从现在出发会访问什么

Forward map `F` 读取当前状态、当前动作以及 prompt `z`，预测从这里开始、之后继续执行 `π_z` 时，未来折扣状态访问分布的向量摘要。

直观地，

`F(s,a,z)^T B(s')`

越大，就表示在当前动作之后继续执行 `π_z`，长期来看越可能访问 `s'`。因此它不是普通的一步 dynamics model `s_{t+1}=f(s_t,a_t)`，而是把**长期 discounted occupancy / successor measure**低秩分解为 `F` 和 `B`。

### `z`：任务方向，而不是动作 latent

每个 `z` 定义一个隐式线性 reward 方向 `r_z(s)=φ(s)^T z`，对应策略 `π_z` 学着长期获得这个方向上的高回报。训练时系统采样许多 `z`，于是同一个 actor 学成一族不同的全身行为。

因此，本文的 `z` 更接近**task/reward prompt**：它在问「希望长期访问哪类状态？」；并不是 LATENT/PULSE 中「decoder 应输出哪种 primitive joint action？」的 latent action。虽然最后两者都能表现为技能，但其训练含义完全不同。

### `π(o_history,z)`：真正执行动作的共享策略

actor 根据可测本体历史和 `z` 直接输出 29 维 PD targets。`F` 和 `B` 负责塑造、评价 prompt 空间，`π` 才是部署时每一步控制机器人的网络。

## 一次完整预训练流程

1. 在 1024 个并行 G1 仿真环境中给每个环境采样一个 latent `z`。
2. 共享策略执行 `a_t=π(o_history,z)`，产生新的机器人状态，并把 transition 存入 replay buffer。
3. 用 replay transition 的 TD/Bellman consistency 训练 `F` 和 `B`，使 `F(s,a,z)^T B(s')`能够近似执行 `π_z` 后对未来状态 `s'` 的折扣访问程度。
4. actor 最大化 `F(s,a,z)^T z`，也就是去实现该 latent 代表的长期行为目标。
5. 仅靠自由探索容易摔倒、卡关节或产生非人动作，所以作者又加入两个 critic：
   - `Q_D`：由 GAN-style discriminator 提供的**人类动作风格**评价；
   - `Q_R`：由关节限位、action rate、自碰撞、脚面方向、打滑等辅助 penalty 构成的**安全/可实现性**评价。
6. actor 最终同时最大化 FB task value、`Q_D` 和 `Q_R`。训练还加入 mass、CoM、friction、joint offset、外力和观测噪声随机化。

训练规模为约 **1.92 亿环境步**、300 万次 gradient updates、UTD=16、batch size 1024。motion data 使用重定向到 G1 的 LAFAN1：40 条、每条数分钟的人体运动。动作数据只提供状态轨迹，**不需要原人形机器人的 actuator action 标签**。

## 重要纠正：预训练阶段没有目标 `s_g`

下面两件事发生在不同阶段，不能混在一起：

```text
预训练阶段：从任务分布 ν 采样 z → 学习一族 π_z 以及 F、B
测试阶段：目标状态 s_g → B(s_g) 得到 z_g → 调用已经训练好的 π_zg
```

预训练时没有人告诉策略「请到达这个特定测试目标 `s_g`」，也不把 `s_g` 送给 `F`。每个并行环境从训练任务分布 `ν` 取得一个 task direction `z`，策略根据当前本体历史和 `z` 输出动作。`B(s_g)` 是所有网络训练完成后才用于指定新目标的 zero-shot goal inference。

因此，一开始策略确实会随机摆动、摔倒或保持不了平衡。这不是算法漏洞：这些 transition 仍被存入 replay buffer；风格 discriminator 和安全 critic 会逐渐学会这些状态不好，FB critic 则从全部 transition 中学习什么状态实际可达。随着 critics 变准，actor 被梯度逐步推向稳定、可控且覆盖多种行为的动作，之后它采集到的数据也越来越好。

## Online、offline、on-policy、off-policy 是两组不同概念

这篇是 **online data collection + off-policy learning**：

| 维度 | 含义 | BFM-Zero |
|---|---|---|
| online / offline | 训练期间是否继续与环境交互、产生新数据 | **online**：策略更新后继续在仿真中采新 transition |
| on-policy / off-policy | 更新当前策略时，能否使用其他/旧策略产生的数据 | **off-policy**：replay buffer 混合了许多旧版本 `π` 的数据，并被重复使用 |

所以，策略每更新一段时间后确实继续采样；这并不会使算法变成 on-policy。On-policy 的关键不是「有没有新数据」，而是更新时要求样本来自当前策略。PPO 基本不能长期复用很久以前的策略数据；BFM-Zero 可以从有限容量 replay buffer 中随机取旧 transition，以当前 actor 产生 bootstrap 动作并更新 `F、B、Q、π`。

buffer 的工作方式近似为：

```text
当前π采一批新数据 → 追加进buffer → 旧数据仍保留
             ↓
从新旧混合数据随机取batch，连续更新16次
             ↓
再用更新后的π采数据；容量满后逐渐淘汰最旧数据
```

只有训练过程中完全不再与环境交互、始终使用固定 transition dataset，才叫 offline RL。本文另有固定的 offline mocap buffer，但控制策略所依赖的 `D_online` 是持续更新的。

## 先复习普通 Bellman 方程：为什么一步数据能学习长期结果

给定策略 `π`，动作价值函数定义为：

`Q^π(s_t,a_t)=E[r_t+γr_{t+1}+γ²r_{t+2}+... ]`。

把第一项拿出来，后面剩余部分就是「从下一状态继续出发的长期回报」，所以：

`Q^π(s_t,a_t)=E[r_t+γQ^π(s_{t+1},a_{t+1})]`，其中 `a_{t+1}~π(s_{t+1})`。

这就是 Bellman 方程。最简记忆是：

```text
当前长期价值 = 眼前立即获得的东西 + γ × 下一时刻剩余的长期价值
```

网络训练时并不知道真实的无限步 `Q`，而是用 target network 构造一步 bootstrap 目标：

`y_t=r_t+γQ̄(s_{t+1},a_{t+1})`。

当前预测与这个目标之间的差：

`δ_t=Q(s_t,a_t)−[r_t+γQ̄(s_{t+1},a_{t+1})]`

叫 TD/Bellman residual。最小化 `δ_t²`，就是要求网络满足 **Bellman consistency**：当前预测不能和「已经发生的即时结果 + 下一时刻预测」互相矛盾。

例如 `γ=0.9`，当前立即奖励为 1，下一状态预测的长期价值为 4.7，那么当前价值应接近 `1+0.9×4.7=5.23`。如果当前网络输出 10，它与下一时刻预测不一致，TD loss 就会把它往 5.23 拉。

Bellman 方法的关键并不是等完整 episode 结束，而是利用每条一步 transition 反复 bootstrap：`C` 的信息先传给 `B`，再传给 `A`，最终用局部的一步数据学习长期结果。

## Successor measure：所谓「长期访问量」到底是什么

先暂时不要看神经网络。给定策略 `π`、当前状态动作 `(s,a)`，定义对任意状态区域 `X` 的 successor measure：

`M^π(X|s,a) = Σ_{k=0}^∞ γ^k Pr(s_{t+k+1}∈X | s_t=s,a_t=a, thereafter π)`。

它表示：执行当前动作后，再一直执行 `π`，未来落入区域 `X` 的**折扣期望访问次数**。它不是「最终状态概率」，也不一定归一化为 1，所以叫 measure 而不是普通 probability distribution。

例如一条不会回头的走廊：

```text
A --right--> B --> C --> terminal，γ=0.9
```

从 A 执行 right：

- 对区域 `{B}`，立即下一步访问一次，`M({B}|A,right)=1`；
- 对区域 `{C}`，第二步访问一次，`M({C}|A,right)=0.9`；
- 对其他不可达位置，访问量为 0。

如果 C 是吸收状态并一直停在那里，则 C 会在后续不断被访问：

`M({C}|A,right)=0.9+0.9²+0.9³+...`。

因此，给定 `(s,a)`，完整的 `M` 就像一张覆盖整个状态空间的「未来会在哪里待多久」的地图。

### Successor-measure Bellman 方程

这张地图可以递归分解：

`M^π(X|s,a) = P(X|s,a) + γ E_{s_1,a_1}[M^π(X|s_1,a_1)]`。

- `P(X|s,a)`：执行 `a` 后，**下一帧立即**落进 `X` 的概率；
- 第二项：到达下一状态 `s_1` 后，按 `π` 继续运动所产生的全部更远未来访问量。

它和普通 value Bellman 的结构完全相同：

```text
普通Q：当前总回报 = 立即reward + γ × 下一状态总回报
Successor measure：当前未来访问地图 = 下一帧状态分布 + γ × 下一状态未来访问地图
```

区别只是普通 Q 是一个标量，而 `M(s,a,·)` 是覆盖所有可能未来状态的巨大函数/分布。

## `s'` 到底指谁：候选未来状态，不是固定终点

论文符号很容易混淆。最好把等式改写成一个新字母 `y`：

`M^{π_z}(dy|s,a) ≈ F(s,a,z)^T B(y) ρ(dy)`。

这里：

- `s,a`：当前起点和当前动作；
- `z`：之后执行哪一个条件策略 `π_z`；
- `y`：我们正在查询的任意候选未来状态；
- `M(dy|s,a)`：未来对 `y` 附近的折扣访问量。

所以 `B` 的输入可以是任何候选状态 `y`：可能是下一帧、五十步后的状态，也可能实际上完全不可达。它不是作者提前给的最终目标。

训练 loss 中同时出现两种状态：

1. `s_{t+1}`：transition 中真实发生的**立即下一帧**；
2. `y=s^+`：从 replay distribution `ρ` 独立抽出的**候选被查询状态**。

论文有些地方都用 prime/plus 表示它们，才显得像同一个量。把它们分开后，FB loss 可简化为：

`[F(s_t,a_t,z)^T B(y) − γ F̄(s_{t+1},a_{t+1},z)^T B̄(y)]² − 2F(s_t,a_t,z)^T B(s_{t+1})`。

- 第一项对任意候选 `y` 施加 successor Bellman consistency；
- 第二个负项把真实下一状态 `s_{t+1}` 注入为「立即访问到的正样本」；
- TD bootstrap 会把这个一步信号不断向更早状态传播，最后形成长期访问关系。

因此不用真的等待轨迹走到一个“最终状态”才训练。每条一步 transition 都提供 immediate successor 信号，TD 负责把多步关系串起来。

## FB loss 逐项推导：`y` 不参与求导

为避免符号混淆，把论文中类似 `Δ(y)` 的写法改成误差函数 `e_FB(y)`：

`e_FB(y)=F(s_t,a_t,z)^TB(y)−γF̄(s_{t+1},a_{t+1},z)^TB̄(y)`。

这里的 `e_FB(y)` 只是「对于候选状态 `y` 的 TD residual」，不是 `dy`、偏导数或 Dirac delta。一次 SGD 更新中：

- transition `(s_t,a_t,s_{t+1})` 从 replay buffer 采样后固定；
- 候选状态 `y` 从 replay state distribution `ρ` 额外采样后固定；
- `z` 固定；
- 对当前网络参数 `θ_F、θ_B` 求梯度，而不是对状态 `y` 求梯度；
- 带横线的 target `F̄、B̄` stop-gradient。

这和监督学习中固定输入图片 `x`、对网络参数求梯度完全一样。`B(y)` 会随 `B` 的权重更新，但输入数据 `y` 本身不被优化。

令：

`m_θ(y)=F(s_t,a_t,z)^TB(y)`，

它是「从当前状态动作出发，未来访问候选状态 `y` 的预测」。理想 successor Bellman target 是：

`m_θ(y) ≈ δ_{s_{t+1}}(y)/ρ(y) + γm̄(y)`。

这里的小写 `δ_{s_{t+1}}` 才是 Dirac delta，表达下一帧确实立即访问了真实 `s_{t+1}`。形式上对上述目标做加权平方回归并展开：

`E_{y~ρ}[(m_θ(y)−γm̄(y)−δ_{s_{t+1}}(y)/ρ(y))²]`，

利用 `E_{y~ρ}[f(y)δ_{s_{t+1}}(y)/ρ(y)]=f(s_{t+1})`，再去掉相对于当前网络为常数的项，可以得到简化目标：

`L_FB = E_{y~ρ}[e_FB(y)²] − 2m_θ(s_{t+1})`。

对应回论文符号：

`L_FB ≈ E_y[(F(s_t,a_t,z)^TB(y)−γF̄(s_{t+1},a_{t+1},z)^TB̄(y))²] − 2F(s_t,a_t,z)^TB(s_{t+1})`。

两部分的作用分别是：

1. `e_FB(y)²`：对任意候选 `y`，让当前 successor 预测与下一时刻 bootstrap 预测满足 Bellman consistency；
2. `−2F^TB(s_{t+1})`：因为 loss 被最小化，这个负项会提高真实下一状态的分数，把「这一步确实访问了谁」注入模型。

第一项还限制整张预测地图，第二项提供真实 positive transition；二者共同作用，预测值才不会无限增大。随着 TD 信号反复传播，`s_{t+1}` 的一步关系逐渐变成两步、三步以及更远未来的访问关系。

## 为什么 `F^T B` 要做内积

想象状态空间只有一万个离散状态。对每个 `(s,a,z)`，完整 successor map 是一个一万维向量：第 `j` 项表示未来访问状态 `j` 的量。把所有起点堆叠起来会得到一个极其巨大的矩阵/算子 `M`。

FB 用 rank-`d` 分解压缩它：

`M ≈ F B^T`。

- `F(s,a,z)` 是当前起点及策略对应的 `d` 维**系数向量**；
- `B(y)` 是候选未来状态对应的 `d` 维**基向量**；
- 两者内积给出巨大矩阵中「从当前起点，在 `π_z` 下访问 y」这一格的近似值。

这和推荐系统很像：用户向量与商品向量的内积近似评分；FB 中则是起点/策略向量与未来状态向量的内积近似 discounted occupancy。这里 `d=256`，用 256 维低秩结构近似无法显式存储的完整未来访问算子。

连续状态下，精确落到一个点的概率通常为零，所以等式实际近似的是相对于 replay state distribution `ρ` 的密度，完整写法带有 `ρ(dy)`。这也解释了为什么数据覆盖不足时，FB 无法可靠表示没见过的区域。

## `B` 的正交约束具体怎么做

若 minibatch 中得到矩阵：

`B_batch = [B(y_1); ...; B(y_n)] ∈ R^{n×d}`，

其中每一行是一个状态的 `d` 维 embedding；每一列则是某一个 latent 特征在全部 `n` 个状态上的响应。把列写成 `B_batch=[b_1,b_2,...,b_d]` 后：

先估计二阶矩：

`C_B = (1/n) B_batch^T B_batch`。

`C_B∈R^{d×d}` 是特征列的 Gram/未中心化二阶矩矩阵，其元素为：

`(C_B)_{kl}=(1/n)b_k^Tb_l=(1/n)Σ_i B_k(y_i)B_l(y_i)`。

目标是：

`C_B ≈ I_d`，即最小化 `||C_B−I_d||_F²`。

含义是：

- 对角线 `(C_B)_{kk}=||b_k||²/n` 接近 1：每个 latent 维度尺度相同且不是全零；
- 非对角线 `(C_B)_{kl}=b_k^Tb_l/n` 接近 0：不同特征列正交，不能全部复制同一特征；
- 若 `n≥d` 且真的达到 `C_B=I`，则 `B_batch` 必然满列秩 `d`；
- 但满秩只是较弱要求。两个特征即使几乎平行，只要没有完全平行仍可满秩，却会数值病态；`C_B≈I` 还要求正交、等尺度和良好条件数。

因此「正交意味着满秩」是对的，但这里真正想要的不只是满秩，而是一个近似 whitening、各向同性、条件良好的状态基底。正交的是 **batch 上的特征列**，不是要求任意两个状态向量 `B(y_i)` 与 `B(y_j)` 都互相正交。

BFM-Zero 附录把它展开成 batch pair 内积形式：惩罚不同样本 `B(y_i)^TB(y_j)` 的平方，同时奖励合适的自身范数；其梯度与令 `E_ρ[BB^T]≈I` 的 whitening/orthonormal loss 对应。

这一点确实和前面 PSG-JEPA / LeWM 的防坍塌思想相似：

| JEPA/SIGReg | FB中的B正交约束 |
|---|---|
| 防止所有图像 latent 变成常数，并鼓励近似各向同性分布 | 防止状态基底坍塌，并让低秩 successor 分解具有完整、稳定的坐标轴 |
| 主任务是预测未来 latent | 主任务是拟合长期 successor measure |
| latent 用于未来表征预测/规划 | `B` 同时作为未来状态基底和 reward/goal 的任务编码器 |

而且如果 `E[BB^T]≈I`，理论中的 state feature `φ(s)=(E[BB^T])^{-1}B(s)` 就近似等于 `B(s)`，因此 goal prompt 才能方便地直接写成 `z_g=B(s_g)`。

## 训练、任务编码和真机控制必须分成三层理解

你的担心有一半是正确的：`B` 的确读取 privileged `(s,o)`，论文也没有训练一个从真实传感器恢复全部 `s` 的 estimator。但它没有在真机闭环中使用 `B(s_real)`；其工程做法是把 **task encoding** 与 **real-time control** 分开。

### 第一层：训练阶段，privileged `s_t` 从哪里来

在线 RL transition 中的 `s_t∈R^{463}` 直接由 Isaac Lab 仿真器给出，包括 root height、body pose、body rotation、各 body 线速度和角速度。它不需要从相机、IMU 或真实机器人估计，因为这一层完全发生在仿真中。

离线 motion buffer 则使用 LAFAN1 人体动作，并把动作 retarget 到 29-DoF Unitree G1。论文直接假设每条 motion trajectory 已包含 `(o_1,s_1,...,o_T,s_T)`：关节位置和 root/body pose 来自重定向后的运动学轨迹，速度可由轨迹的相邻帧得到。论文说明了数据集、G1 retargeting 和所含状态，但没有在正文中详细展开一套独立的 mocap-to-privileged-state 算法；官方仓库直接提供处理后的 `lafan_29dof*.pkl`。

因此训练时的 privileged state 来源是：

```text
在线 policy 数据：Isaac Lab 真值
专家 motion 数据：LAFAN1 → G1 retargeting → 预处理状态轨迹
```

### 第二层：任务编码阶段，`B` 仍然可能需要 privileged task description

这里必须比「部署不需要 `B`」说得更精确：

- 如果使用作者已经生成好的 prompt，`B` 完全不需要部署；
- 如果要添加一个新的 pose、reference motion 或 state reward，通常仍需在场外运行 `B` 来生成新的 `z`；
- 此时 `B` 读取的是**已知目标/参考/仿真样本的完整状态**，不是读取真实机器人当前的 privileged state。

三类任务的具体来源为：

1. **Pose goal**：论文从 retargeted LAFAN1/AMASS 状态中抽取目标 pose，并把速度分量清零，得到已知的完整 `(s_g,o_g)`，场外计算 `z_g=B(s_g,o_g)`。
2. **Motion tracking**：reference motion 本身就是一段预先已知、已 retarget 的完整状态序列。场外计算每一帧或 look-ahead window 的 `B(reference)`，得到一串 `z_t`。论文真机 tracking look-ahead 为 3。
3. **Reward inference**：从仿真训练 replay buffer 抽约 40 万个 privileged states，在这些状态上计算人工 reward `r(s_i)`，再求 `z_r≈(1/N)Σ_i r(s_i)B(s_i)`。真机不重新计算 reward，也不测量对应 privileged state。

作者还展示了由单目真人视频估计动作再 retarget 到 G1，但这仍然是**场外制作 reference sequence**；不是机器人机载视觉在线理解视频或环境。

### 第三层：真实 G1 的 50 Hz 闭环只运行 actor

官方部署代码进一步确认了这一点。部署包把模型组织成：

```text
model.onnx                         # 实时 actor policy
tracking_inference/*.pkl          # 预计算 z 序列
goal_inference/*.pkl              # 预计算 goal z
reward_inference/*.pkl            # 预计算 reward z
```

这些文件复制到 Unitree G1 的 Jetson Orin。运行时通过 Unitree SDK 以 50 Hz 读取：

- 29 维关节位置；
- 29 维关节速度；
- root angular velocity；
- projected gravity；
- 最近 4 步本体观测与历史动作；
- 从 `.pkl` 读取的固定或时变 256 维 `z_t`。

实时循环是：

`a_t=π_ONNX(o_{t,H},z_t) → 29维PD position targets`。

不需要实时运行 `F、B、D、Q_D、Q_R`，也没有输入真实 root position、root linear velocity 或所有 body 的完整 pose/velocity。官方部署说明报告 Jetson 推理延迟约 4–5 ms。

### 所以它到底鸡不鸡肋：准确判断

如果任务是作者展示的 motion tracking、已知目标姿态和预定义 state reward，它不是鸡肋：完整 reference 或仿真 reward samples 本来就已知，提前转成 `z` 后，真机 actor 确实只靠本体传感器运行；这也是 asymmetric actor-critic 的正常使用方式。

但它的“promptable”范围确实比语言/视觉 foundation model 窄：

- 不能从真实场景中的未知物体、目标位置或视觉观测自动构造 privileged `s_g`；
- 没有 language encoder。论文所说 reward formulation 对 language prompt 友好，只能理解为未来可由语言模型选择/组合人工 reward，并未实际实现语言到 `z`；
- 对新任务，用户仍必须提供可构造完整状态的 reference、能在仿真 state 上计算的 reward，或另外开发 perception/task encoder；
- reward inference 依赖 replay buffer 覆盖相应状态，超出数据覆盖的真实任务不能靠公式凭空得到可靠 `z`。

因此最准确的结论是：**BFM-Zero 消除了真机闭环对 privileged current state 的依赖，但没有消除任务定义阶段对 privileged task representation 的依赖。**它解决的是通用本体运动控制和任务复用，不是开放世界的任务理解与感知。

## `z_motion` 为什么归一化，和 rollout 的 `z` 是不是同一个

`z` 是通用变量名；`z_motion` 或 `z_τ` 是它的一种具体构造方式：

`z_τ = mean_{s∈τ} B(s)`，随后做 `z_τ ← √d · z_τ/||z_τ||_2`。

FB 的任务空间通常取半径 `√d` 的超球面。归一化有三个作用：

1. 平均后的范数会受轨迹长度、状态重复和向量抵消影响；归一化去掉这种无关尺度；
2. reward 乘正常数不会改变最优策略，所以任务向量主要需要表达**方向**而不是大小；
3. 让 motion-derived prompt 与 uniform-sphere prompt 范数一致，避免 discriminator 仅靠 `||z||`就判断 expert/policy。

继承自 FB-CPR 的训练分布 `ν` 实际混合三种 `z`：

- `z_τ`：由 mocap trajectory 编码的 motion prompt；
- `B(s)`：由 online replay 状态形成的 goal-reaching prompt；
- 超球面均匀采样的随机 `z`：保证更广的任务覆盖。

在 discriminator 数据中：

```text
正样本：(expert trajectory中的状态 s_e, 该轨迹的 z_τ)
负样本：(π_{z_i} rollout访问的状态 s_i, 生成这条rollout的 z_i)
```

`z_i` 与 `z_τ` 属于**同一个 latent space、同一维数和相同范数约定**，但通常不是数值相等。若某次 rollout 正好使用由某条 motion 生成的 `z_i=z_τ`，discriminator就在比较「这条条件策略产生的状态是否覆盖该 motion 的状态」；若 `z_i` 来自 goal或随机方向，它仍参与 policy joint distribution `p_π(s,z)`的匹配与FB探索。

所以 discriminator 不是简单学习“expert state永远为真，policy state永远为假”，而是在匹配两个**联合分布**：

`p_M(s,z)`：motion数据中的状态与对应motion prompt；

`p_π(s,z)`：条件策略 `π_z`实际访问的状态与其prompt。

目标是让同一个 `z` 条件下，策略访问的状态分布接近对应的数据行为分布；归一化和混合 `ν` 也能减少 `D` 只看 `z` 范数作弊的可能。

## 所有训练网络：输入、输出、loss 与部署去向

令 `o_H` 表示 actor 可测的本体与动作历史，`s` 表示只在仿真训练时可见的完整 privileged state，`x=(o_H,s)`。

| 网络 | 输入 → 输出 | 怎样训练 | 真机实时控制是否需要 |
|---|---|---|---|
| Actor `π_θ` | `(o_H,z) → 29维PD target a` | 最大化 `F^Tz + α_D Q_D + α_R Q_R`，即 deterministic policy gradient | **需要** |
| Forward critic `F_ψ` | `(x,a,z) → 256维向量` | FB successor-measure TD loss | 不需要 |
| Backward map `B_ω` | `(s,o) → 256维状态特征` | 与 `F` 联合优化的 FB loss和正交/防坍塌项 | 真机控制循环不需要；生成 goal/motion/reward prompt 时使用 |
| Discriminator `D_η` | `(s,o,z) → expert-like概率` | expert mocap 与 policy states 的二分类 BCE/GAN loss | 不需要 |
| Style critic `Q_D` | `(x,a,z) → 标量` | 用 `D` 产生的风格 reward 做 TD learning | 不需要 |
| Safety critic `Q_R` | `(x,a,z) → 标量` | 用关节限位、打滑、自碰撞等 auxiliary reward 做 TD learning | 不需要 |
| Target networks | `F、B、Q_D、Q_R` 的慢更新副本 | soft/target update，为 TD target 提供稳定参照 | 不需要 |

论文对 `F、Q_D、Q_R` 使用两个 critic ensemble；actor 是一个约 31.9M 参数网络。训练系统总计约 440.5M 参数，但部署不等于运行全部 440.5M。

## 一次交互和一次梯度更新，逐步发生什么

### A. 先与仿真交互：此时还没有梯度

对 1024 个并行环境分别采样一个 256 维 `z_e`。在一段 rollout 内，环境 `e` 使用：

`a_t = π_θ(o_{t,H},z_e)`。

`a_t` 是 29 维 PD position target。仿真器利用真实模拟动力学得到下一状态 `(o_{t+1,H},s_{t+1})`。随后保存：

`(o_{t,H}, s_t, a_t, o_{t+1,H}, s_{t+1}, z_e)`。

这里不需要动作成功，也没有 `s_g` 或 goal reward。环境完成的只是产生经验数据；**不能把梯度穿过 Isaac Lab 动力学传回来**。

### B. 从 replay buffer 取 batch

训练器随机取出 1024 条旧 transition。off-policy 的含义就在这里：数据未必由当前最新 actor 生成，可以被重复使用。每个并行环境步会做 16 次网络更新，即 UTD=16。

与此同时，从固定 mocap buffer 取若干长度为 8 的人体运动状态序列。mocap 没有机器人 action label，主要供风格 discriminator 使用。

### C. 更新 `F` 与 `B`：学习长期可达关系

理想目标是：

`F(x_t,a_t,z)^T B(x') ≈ 从(x_t,a_t)开始、之后执行π_z时访问x'的折扣概率`。

可以把 successor measure 的 Bellman 关系简化成：

`当前的长期访问量 = 立即到达下一状态 + γ × 下一时刻的长期访问量`。

FB loss 用 batch 内状态作为许多候选 `x'`：

- 对真实配对的下一状态 `x_{t+1}`，提高 `F(x_t,a_t,z)^T B(x_{t+1})`；
- 对其他 batch 状态，利用 TD residual 使当前预测与 target `F̄(x_{t+1},a_{t+1},z)^T B̄(x')`一致；
- 对 `B` 加正交/归一化项，防止所有状态都变成同一个向量；
- 另一个 consistency 项使 `F(x,a,z)^Tz`满足对应 successor-feature Q 的 Bellman 关系。

最终得到标量 `L_FB(ψ,ω)`，普通反向传播计算：

`ψ ← ψ − lr_F ∇_ψ L_FB`

`ω ← ω − lr_B ∇_ω L_FB`。

这一步更新 `F` 和 `B`，**不直接更新 actor**。带横线的 target `F̄、B̄` 在计算 TD target 时 stop-gradient，避免目标和预测同时快速移动。

### D. 更新 discriminator `D`：什么样的行为像人

对一段 mocap 状态序列，先用当前 `B` 编码并平均：

`z_τ = normalize(mean_t B(s_t^expert,o_t^expert))`。

然后让 `D` 区分：

- `(expert state, z_τ)`：标签为 expert；
- `(policy state, rollout时的z)`：标签为 policy-generated。

使用标准 binary cross-entropy / GAN discriminator loss 更新 `D`：

`η ← η − lr_D ∇_η L_D`。

再把判别结果转换成风格 reward，例如 log-odds：

`r_D = log D(x,z) − log(1−D(x,z))`。

策略状态越符合与该 `z` 对应的人类动作风格，`r_D` 越高。

### E. 更新两个普通标量 critic

风格 critic 学累计风格回报：

`y_D = r_D + γ Q̄_D(x_{t+1},π(o_{t+1},z),z)`

`L_QD = (Q_D(x_t,a_t,z)−y_D)^2`。

安全 critic 学累计 auxiliary rewards：

`y_R = Σ_k r_aux,k + γ Q̄_R(x_{t+1},π(o_{t+1},z),z)`

`L_QR = (Q_R(x_t,a_t,z)−y_R)^2`。

然后分别用 MSE 梯度下降更新 `Q_D、Q_R`。这里的 auxiliary rewards 包括关节限位、action rate、自碰撞、脚部方向、ankle roll 和 feet slip。论文表中这些主要是 penalty，因此 `Q_R` 越大通常代表长期越少违反约束。

### F. 最后更新策略网络 `π_θ`

从 batch 的本体历史和 `z` 重新让**当前 actor**产生动作：

`u_i = π_θ(o_{i,H},z_i)`。

把这个动作送入已经训练到当前水平的三个 critic，组成 actor value：

`V_actor = F(x_i,u_i,z_i)^Tz_i + α_D Q_D(x_i,u_i,z_i) + α_R Q_R(x_i,u_i,z_i)`。

三部分分别代表：

1. `F^Tz`：是否实现该 latent direction 对应的长期行为；
2. `Q_D`：动作后续是否接近人类 motion style；
3. `Q_R`：动作后续是否满足物理/安全约束。

actor loss 取负号：

`L_actor = −mean(V_actor)`。

在 actor 更新这一步，`F、Q_D、Q_R` 的参数视为固定；但它们对动作输入 `u` 是可微的。梯度按链式法则穿过 critic 的动作输入回到 actor：

`∇_θ L_actor = −(∂V_actor/∂u)(∂π_θ(o,z)/∂θ)`。

因此优化器执行：

`θ ← θ − lr_π ∇_θ L_actor`。

换成直觉就是：critic 告诉 actor「如果把当前 29 维动作往哪个方向调整，长期 task value、人类风格和安全性会提高」；actor 沿这个方向更新参数。它不需要环境可微，也不使用 PPO 的 advantage、ratio 或 clipped objective。这就是 deterministic/off-policy policy gradient 的核心。

最后慢更新 target networks，再继续采样、更新。整个循环是：

```text
较差的初始π产生杂乱/摔倒数据
        ↓
F/B从transition学习可达结构；D和QR学风格与安全
        ↓
critic对动作的梯度改善π
        ↓
改善后的π产生更丰富、更稳定的数据
        ↓
继续迭代
```

随机 `z` 为什么不会全部学成同一种动作？因为 `B` 的正交约束使不同状态方向不会全部坍塌，actor 对每个 `z` 最大化不同的内积 `F^Tz`；不同方向会偏好不同的长期状态访问模式。motion discriminator 和 safety critic再把这些模式限制在人形机器人可实现、相对自然的范围中。

## 训练完成后，`s_g` 到底是什么

论文的 pose-reaching 目标不是空间中的一个 `(x,y,z)` 点，而是从 LAFAN1/AMASS 动作中抽出的**完整稳定机器人状态**：

- 29 维关节位置/身体构型；
- root height、各 body pose/rotation；
- 线速度与角速度设为零；
- 对应的 observable proprioceptive components。

论文写 `z_g=B(s_g)` 是简化记号；其实现中的 `B` 输入包括 `(s_g,o_g)`。评估时主要用 29 维关节位置误差衡量是否接近目标姿势。

测试流程才是：

```text
完整目标姿势 s_g
        ↓ 训练好的B，只算一次
z_g = B(s_g)
        ↓
训练好的 π(o_history,z_g)
        ↓
29维PD target → 仿真/真机
```

这里是 `B` 编码目标，不是 `F`。`F` 的作用是在**预训练阶段**估计执行动作后长期会访问哪些状态，并为 actor 提供 `F^Tz` 的梯度。

为什么 `z_g=B(s_g)` 会促使机器人走向目标？粗略地说，`F` 表示未来访问状态的 `B` 特征累计；当 actor 最大化 `F^TB(s_g)` 时，它偏好那些未来状态特征与目标状态 `B(s_g)`高度对齐的动作。经过 FB 训练和 `B` 的正交结构，这就近似于最大化未来访问目标状态附近区域的程度。

特别要注意：actor 真机运行时没有 privileged `s_t`，它只接收本体历史和固定/更新的 `z_g`。完整 `s_g` 只是离线产生 prompt；不是每一帧都把目标完整状态与当前状态一起送进 policy。

### 为什么说「无监督 RL」却又有 mocap 和 reward？

这里的 unsupervised 更准确地说是：**预训练没有给定下游任务 reward**，例如“向前走 0.7 m/s”“跟踪这段舞蹈”或“到达这个姿态”。它并不是完全无监督、无偏置：

- mocap 轨迹被用来训练专家/策略 discriminator，提供 human-like style reward；
- 手工 safety regularization rewards 限制危险动作；
- 仿真完整状态作为 privileged supervision；
- 最终行为范围仍明显受到 LAFAN1 motion coverage 影响。

所以不能概括为「不给任何 reward，机器人自己学会所有动作」。更准确的是：**不提供具体下游任务奖励，以 successor representation 学通用任务空间，同时用 mocap 风格和安全奖励约束探索。**

## 训练后，三种 prompt 怎么变成同一个 `z`

### 单个目标姿态

给定目标完整状态 `s_g`：

`z_goal = B(s_g)`。

然后固定或顺序切换这些 `z_goal`，actor 会寻找能长期到达并维持相应状态的动作。它没有逐时刻 reference，也没有另外训练 pose-reaching policy。

### 一整段 reference motion

对当前时刻以后一个短 look-ahead window 内的 reference states 做 `B` 编码并求和：

`z_t = Σ B(s_{t:t+H})`。

每个时刻更新 `z_t`，同一个 actor 就成为 motion tracker。真实 G1 上 look-ahead 为 3。这里仍需持续提供参考动作片段，只是无需为这段动作重新训练 tracker。

### 任意状态 reward

从训练 replay buffer 取许多状态，对每个状态算用户给定 reward `r(s_i)`：

`z_reward ≈ (1/N) Σ B(s_i) r(s_i)`。

论文推理时使用约 40 万个样本。它相当于将 reward 投影到已经学到的 FB task basis，再由 `π_z` 执行。例如可以写「站立且前进 0.7 m/s」「降低 pelvis」「抬高手腕」，也能把几个 reward 线性组合。

这不是自然语言 prompt：用户必须写出能在机器人**状态**上计算的 reward，而且 replay buffer 必须覆盖相应状态。若目标行为超出表示与数据覆盖，公式不保证找到正确 `z`。

## Zero-shot 和 few-shot 分别是什么意思

- **Zero-shot**：预训练结束后，把目标姿态、reference motion 或 reward 直接转换成 `z`；不更新 actor 参数，也不为任务与仿真/真机额外交互。
- **Few-shot adaptation**：当直接算出的 `z` 不够好时，在仿真中以 zero-shot `z` 为起点搜索更好的 prompt。单姿态用 CEM；整段动作优化一串 `z_t`，使用类似 DIAL-MPC 的采样优化。依然不微调神经网络权重。

例如给 G1 torso 增加 4 kg 负载后，原本单腿站立 prompt 在 5 秒内失稳；用 CEM 搜索 20 轮得到新 `z` 后，真机保持超过 15 秒。跳跃轨迹在改变地面摩擦后，通过 latent trajectory optimization 将跟踪误差降低约 29.1%。这里的 few-shot 数据和搜索发生在仿真，不是真机边运行边梯度更新。

### 关键澄清：`z` 最初来自 `B`，为什么适配时却不更新 `B`

对 pose goal，zero-shot 起点确实由 backward map 给出：

`z^(0)=B(s_g,o_g)`。

但 `B(s_g)` 只是把目标状态转换成一个合理的初始 task direction，并不要求之后所有有效 prompt 都必须严格等于某个状态的 `B(s)`。Few-shot 阶段把 256 维 `z` 本身当成自由决策变量：

```text
完整目标状态(s_g,o_g)
        ↓ 固定的B，只编码一次
zero-shot prompt z^(0)
        ↓ CEM / sampling-based latent optimization
z^(1) → z^(2) → ... → z*
        ↓
固定actor π(o_H,z*)部署
```

搜索时冻结全部网络：`π、F、B、D、Q_D、Q_R`。每个候选 `z` 被送入同一个固定 actor，在仿真中 rollout，再用显式任务目标与安全 penalty 评价：

`J(z)=Σ_t[r_task(s_t)−α_RΣ_k r_aux,k(s_t,a_t)]`。

单姿态适配用 CEM：围绕 `z^(0)`采样候选 latent，仿真评价，保留 elite samples，用 elite 更新 latent 采样分布，重复 20 轮得到 `z*`。整段 tracking 则以 `B(reference)`产生的 zero-shot `z_t` 序列为 warm start，用类似 DIAL-MPC 的 dual-annealing sampling 同时搜索一串 latent prompts。

因此它不是重新加载 checkpoint 继续训练，也不是 PPO，更没有 on-policy/off-policy 的策略更新之分。CEM 是不求梯度的黑盒 prompt optimization：网络参数不动，只有输入向量 `z` 的数值变化。

为什么 `z*` 可以偏离 `B(s_g)`？直觉上：

`z* ≈ B(s_g)+Δz_balance+Δz_payload+Δz_safety`。

这是解释性的写法，论文并没有显式分解这些分量。它表示原始目标 pose prompt 只描述「想去哪里」，搜索得到的偏移还可能吸收负载变化、平衡需求、仿真动力学和安全约束。最终 `z*` 不必对应一个单独状态，而可以是 latent task directions 的组合。

所以论文中 post-training 的准确含义是：**`B` 提供 zero-shot warm start，随后冻结整个 BFM，直接做 soft-prompt / latent-prompt tuning。**论文引言提到未来也可 fine-tune，但实验真正验证的是 prompt-level optimization，并没有展示 resume 原来的 off-policy FB 训练或改用 PPO 微调网络。

## 实验做了什么

### 仿真量化

- 在 LAFAN1 上评估 motion tracking、21 个静态 pose reaching 和 6 类人工 reward tasks。
- 在 MuJoCo 中进行 sim-to-sim 检验，Isaac 与 MuJoCo 的指标变化均小于约 7%。
- 使用 AMASS-CMU 的 175 条未见 motion 和 10 个姿态检查 OOD tracking/goal reaching。
- 相比无 DR、所有网络都看 privileged state 的理想版，真实可部署版在 tracking、reward、pose 三类指标上分别下降约 2.47%、25.86%、10.65%；reward inference 对 domain randomization 后的数据最敏感。
- 附录考察了 motion dataset 和网络规模，但这些 scaling 消融只跑了一个 seed，不能据此得出稳健 scaling law。

需要注意：论文主要量化比较的是自身 privileged 版本、domain randomization、跨仿真器和 OOD 数据，**没有在统一协议下与强 task-specific PPO motion tracker 做完整成功率/跟踪精度对比**。因此它最强的证据是「一个 promptable policy 确实覆盖多种任务并可真机部署」，而不是证明每个单项任务都超过专门训练的 PPO SOTA。

### 真实 Unitree G1

同一个 actor 展示了：

- 舞蹈、格斗、球类、风格化行走等 motion tracking；
- 从不同初始姿态到 T-pose 等单姿态 goal reaching；
- 根据显式 reward 实现前后/侧向行走、旋转、蹲坐、低姿态移动和抬臂；
- reward 相加得到「后退同时抬臂」等组合技能；
- 受推、被踢甚至被拉倒后的自然恢复；
- 4 kg 负载下经仿真 prompt adaptation 后的单腿站立。

真机部分很有说服力地展示了行为覆盖与鲁棒性，但论文明确把它定位为**定性验证**：选取若干任务、所有结果来自一个模型，没有给出大规模真机成功率、与 PPO baseline 的真机对比或统计置信区间。不能因为视频丰富就写成定量全面超越现有 tracker。

## 它和 LATENT/PULSE 路线到底差在哪

| 问题 | LATENT / PULSE 式 latent action | BFM-Zero |
|---|---|---|
| 基础训练 | PPO tracker 先学 reference tracking，再 online distillation / VAE | 不先训 reference tracker；FB-CPR 直接训练 `π(o,z)、F、B` |
| `z` 的含义 | decoder 要执行的低维动作/primitive code | task/reward direction，指定希望策略长期访问什么状态 |
| motion data 用法 | reference 输入 tracker/posterior，蒸馏 teacher action | action-free mocap state trajectory训练风格 discriminator，并可经 `B` 形成 prompt |
| 下游任务 | 高层 PPO 再根据球状态选择 latent，主要解决网球任务 | goal、motion、reward 直接转换成 `z`，通常不再训练高层 policy |
| 输出动作 | decoder 先从 `(state,z)` 解码关节 target | actor `π(history,z)`直接输出 29 维 PD target |
| 适配 | 训练新的 high-level task policy | 可直接算 `z`，困难时在仿真里搜索 `z` 或 `z` 序列 |

最简记忆：**LATENT 的 latent 更像低层技能动作接口；BFM-Zero 的 latent 更像长期行为目标接口。**LATENT 仍需 PPO 学会在网球状态下如何选技能；BFM-Zero 试图让新目标本身通过 FB 表征直接变成 policy prompt。

## 真正的创新点

1. 将过去主要用于虚拟 humanoid 的 FB / unsupervised RL 策略族，结合 history actor、privileged critics、domain randomization 和安全 reward，做到真实 G1 sim-to-real。
2. 用同一个 256 维 latent space 统一表示 goal、motion sequence 和 state-based reward，并让同一 actor 在三类任务之间切换，无需重新训练。
3. 证明 motion dataset 不需要 actuator action label：状态轨迹主要用于风格约束和 prompt 构造，而不是 teacher action imitation。
4. 在 zero-shot 不足时只优化 prompt，不动 440M 级训练网络，并展示负载/摩擦变化下的快速适配。

算法的 FB 基础并非本文从零提出，核心直接继承 FB-CPR；本文更重要的新增贡献是**把这一 off-policy unsupervised RL 范式改造成可真实部署的人形 BFM，并完成工程验证**。

## 局限与我的判断

- 没有视觉、地形感知、物体状态或语言输入；这里的 foundation 主要指**全身本体运动控制任务族**，不是能理解开放世界指令的通用 humanoid foundation model。
- reward prompt 必须由状态显式计算，并依赖约 40 万 replay states 做投影；稀疏 reward 和 domain-randomized buffer 已在实验中出现不稳定结果。
- tracking 的“零样本”仍需每时刻的 reference look-ahead；只是不用针对该 motion 重新训练，不等于机器人自己理解视频意图。
- 预训练并非纯 reward-free，motion-style discriminator 与手工安全项对可部署性至关重要。
- 440.5M 训练模型、1.92 亿环境步和高 UTD replay 更新的训练成本不小；off-policy 的样本复用优势不等于计算便宜。
- 行为覆盖仍受 LAFAN1 数据和仿真可达状态限制。作者也承认更复杂动作需要更可靠的在线适应。
- 真机结果主要为定性展示，缺少统一的强 baseline 和重复统计。

**最终判断**：这篇的重要性不在于比 PPO 更会跟踪某一条 motion，而在于改变了任务组织方式。PPO tracker 把 reward/reference 固定在训练中；BFM-Zero 先用 FB 学一个长期行为基底，之后再把目标、轨迹或 reward 投影成 `z`。这是目前人形 BFM 方向中很有代表性的思路，而且真实 G1 证明 off-policy unsupervised RL 并非只能在 character simulation 中工作。但它还只是本体状态级的 behavioral foundation model，尚未覆盖视觉感知、场景交互和 loco-manipulation 的开放任务。

# SONIC: Supersizing Motion Tracking for Natural Humanoid Whole-Body Control

- **时间/发表**：arXiv v1 为 2025-11，当前 v4 为 2026-08；正式发表于 **Science Robotics 11(117), eaed4592 (2026)**。
- **作者/单位**：Zhengyi Luo、Ye Yuan、Tingwu Wang 等，NVIDIA。
- **机器人/仿真**：29-DoF Unitree G1；Isaac Lab 大规模 PPO 训练、MuJoCo sim-to-sim、真实 G1 部署。
- **规模**：源动作约 700 小时，过滤后 611 小时、超过 1 亿帧/317,189 clips；最大模型 42M 参数；128 GPUs、约 21k GPU hours、7 天。
- **代码/模型/数据**：[论文 v4](https://arxiv.org/html/2511.07820v4) · [Science Robotics DOI](https://doi.org/10.1126/scirobotics.aed4592) · [项目页](https://nvlabs.github.io/GEAR-SONIC/) · [代码](https://github.com/NVlabs/GR00T-WholeBodyControl) · [GEAR-SONIC 模型](https://huggingface.co/nvidia/GEAR-SONIC)。

## 一句话总结

SONIC 没有像 BFM-Zero 那样用无监督 FB 学 reward/task direction，而是把一个**显式 reference motion tracker**用 PPO 扩展到 611 小时人体动作，并把 robot reference、human SMPL motion 和 sparse teleoperation command 都编码为统一的 64 维离散 motion token。统一 control decoder 再根据 token 与真机本体历史输出 29 维关节位置目标；上层 kinematic planner、VR、视频/文字/音乐 motion generator 或 VLA 都可以通过同一 token 接口调用这个低层控制基础模型。

## 它是不是 foundation model

**可以把它称为 humanoid whole-body motion/control foundation model，但必须限定 scope。**它具有 foundation-like 的几个核心特征：

- 数据、模型和计算规模明显大于传统单任务 humanoid controller；
- 同一个策略覆盖行走、跑、跳、舞蹈、格斗、爬行、下蹲、工具动作等大量行为；
- 在未见动作和不同动作来源上具有 zero-shot tracking generalization；
- 提供统一 token/action interface，可被 planner、teleoperation 和 VLA 复用；
- 真实 G1 上进行了较大规模验证，而不是只展示少量 cherry-picked clips。

但它不是完整的视觉-语言-动作 foundation model：SONIC controller 本身不读取 RGB 或自然语言，也不做场景语义理解。文字/音乐/视频先由外部 GEM 生成人体 motion，VLA loco-manipulation 则由另一个 GR00T N1.5 模型读取视觉语言并预测 SONIC token。因此更准确的定位是：

> **SONIC 是一个 motion-conditioned 低层 whole-body control foundation，以及供上层模型调用的运动 action tokenizer/decoder。**

## 核心任务仍然是什么

预训练任务非常传统：给定未来 reference motion，控制 G1 在物理仿真中跟踪它。

状态由两部分构成：

1. `s_p`：最近 10 步关节位置、关节速度、root angular velocity、projected gravity 和历史动作；
2. `s_g`：未来 motion command，可以来自 robot reference、human motion 或 hybrid teleoperation command。

动作是：

`a_t∈R^29 = 29个关节的 target positions`，

再由关节 PD controller 转为电机力矩。它不是直接 torque control。

奖励仍然是显式 tracking reward：root position/orientation、各 body link 的相对位置/方向/线角速度、头/手腕/脚踝 end-effector 位置，加上 action rate、joint limit、undesired contact、anti-shake 和 feet acceleration penalties。

因此论文说 motion tracking “without manual reward engineering”，更准确应理解为：**不再为跳舞、格斗、爬行等每一种行为分别设计任务 reward；但仍然有一套人工设计且较复杂的通用 motion-tracking reward。**

## 数据怎么得到

源数据是约 700 小时大规模人体 mocap，包含日常动作、locomotion、舞蹈、gesture、combat、tool use、object manipulation、受伤步态和风格化动作等。作者使用 GMR 与 PyRoki 将人体 motion retarget 到 Unitree G1，并过滤楼梯、坐椅等对该 G1 配置物理上不合适的动作，剩下：

- 611 小时训练数据；
- 50 Hz、超过 1 亿帧；
- 317,189 clips；
- 33 个 main categories、8,447 个训练 sub-categories。

其中 288 小时、142,220 条 motion 以 BONES-SEED 数据集公开。训练不仅保留 G1 reference，还构造相互同步的三种 command 表示，使同一段语义动作同时拥有 robot、human 与 hybrid view。

### 这些 human motion 到底要不要 retarget

要区分 **训练数据准备** 和 **训练好后的在线输入**：

1. **训练阶段仍然需要显式 retargeting。**原始 SMPL/人体 mocap 不能直接充当 G1 的关节 reference。作者先用 GMR/PyRoki 把人体动作离线 retarget 到 29-DoF G1，得到同步的 `(human motion, robot motion)` 配对数据。robot encoder、tracking reward 和 reconstruction target 都依赖这条 robot-space reference。
2. **训练好后，human encoder 可以省掉在线 retargeting。**部署时新的 SMPL motion 可以直接经过 `E_h → token → D_c`；因为 `E_h` 已用上述配对数据学会将 human representation 翻译到 G1 可执行的 token。因此这不是“不需要 retargeting”，而是把过去显式逐帧求解的 retargeting 蒸馏进了网络。
3. **robot encoder 仍要求 robot-space motion。**如果上层本来就输出 G1 reference，就走 `E_r`；如果输入是人类动作，才走 `E_h` 的 learned retargeting。
4. **hybrid encoder 是两者混合。**稀疏的头和双手来自 human/VR 空间，未来腿部 reference 已经是 robot-space motion；因此它也不是完全无 retargeting。
5. **VLA 直接预测 token 时没有 runtime retargeting。**此时 VLA 学的是 SONIC token 这一动作接口，而不是先生成 SMPL 再转换。

所以最准确的结论是：

> SONIC 没有消灭 embodiment mapping，而是用离线 retargeting 构造监督，再把 human-to-robot mapping 隐式学进 `E_h`；这让在线输入人体 motion 更方便，但能力边界仍受训练时 retargeting 数据质量约束。

## 三个 encoder 在编码什么

SONIC 不是一个 encoder 处理所有输入，而是三个专门 MLP 映射到同一个 token space：

### Robot motion encoder `E_r`

输入未来 10 帧 G1 reference joint positions/velocities，帧间隔 0.1 s。它最接近普通 motion tracking：reference 已经 retarget 到 G1。

### Human motion encoder `E_h`

输入未来 10 帧人体 3D joint/SMPL motion，帧间隔 0.02 s。它允许视频、文字、音乐或 VR 生成的人体动作绕过显式 runtime retargeting，直接变成统一 token。

### Hybrid encoder `E_m`

输入当前帧 sparse upper-body keypoints（头、双手）以及未来 lower-body robot motion。它适合只有头显和手柄的 3-point teleoperation：上半身追踪操作者，腿部由 planner 生成。

三个 encoder hidden dimensions 均为 `[2048,1024,512,512]`。它们的目的不是分别学三套 policy，而是把同一个动作的三种描述对齐到共同的 motion representation。

## Universal token 和 FSQ 是什么

encoder 先输出连续 latent，再经过 **Finite Scalar Quantization（FSQ）**离散化。默认配置使用两个 token，每个 32 维，每一维有 32 个量化 level，因此控制接口展开后是 64 维 token。

FSQ 可以直观理解为：

```text
连续motion latent
       ↓ 每个标量舍入到有限level
两个32维离散token = 64维universal motion token z
```

它与 VQ-VAE 的主要区别是没有需要学习/维护的离散 codebook，因此避免大量 code 不被使用的 codebook collapse，也更方便用 straight-through estimator 与 PPO 联合反向传播。

这里的 token 不是语言 token，也不是 BFM-Zero 的 reward/task direction。它编码的是**接下来短时间希望身体怎样运动**，更像物理可执行的 short-horizon motion command / action primitive interface。

## 两个 decoder 分别做什么

### Control decoder `D_c`

读取统一 token 和当前 10 步本体历史：

`a_t=D_c(z,s_p)`，

直接输出 29 维关节 target positions。该网络是部署时真正的低层 policy，hidden dimensions 为 `[4096,4096,2048,2048,1024,1024,512,512]`。

### Robot motion decoder `D_r`

只读取 token，重建对应的 robot reference：

`ĝ_r=D_r(z)`。

它主要是训练时的 auxiliary decoder，不负责真机逐步控制。特别是输入为 human motion 时，`E_h → z → D_r` 本身构成一个 learned human-to-robot retargeting path。

## 一次完整训练流程

```text
同一段动作的同步表示
 ├─ robot reference g_r → E_r → z_r
 ├─ human/SMPL g_h     → E_h → z_h
 └─ hybrid command g_m → E_m → z_m
                                ↓ FSQ
                  unified discrete motion token
                        ├─ D_c(token, proprio) → 29D action
                        └─ D_r(token) → reconstructed robot reference
```

总 loss 为：

`L=L_PPO+L_recon+L_token+L_cycle`。

### `L_PPO`：让 token 真正物理可执行

在 Isaac Lab 中运行 G1，根据 tracking reward 用 on-policy PPO 同时更新 encoder、FSQ、control decoder 和 critic。也就是说 token space 不是单独训好 AE 再冻结，而是被 physics-based RL 直接塑造成 control-useful representation。

### `L_recon`：三个输入都能恢复同一个 robot motion

`||D_r(z_r)−g_r||² + ||D_r(z_h)−g_r||² + ||D_r(z_m)−g_r||²`。

无论输入是 robot reference、人体动作还是 sparse hybrid command，都应恢复相应 G1 reference。

### `L_token`：三种 encoder 对齐

`||z_r−z_h||²+||z_r−z_m||²+||z_m−z_h||²`。

同一动作的三种描述应落到相近 token，否则切换输入接口时 control decoder 会看到三个不一致的空间。

### `L_cycle`：human token 翻译成 robot motion 后仍保持语义

`||E_r(D_r(z_h))−z_r||²`。

先把 human token 解码成 robot reference，再重新编码，应该回到对应 robot token。它进一步约束 human-to-robot translation 不丢失核心 motion information。

所有 loss 在一个 end-to-end loop 联合优化。PPO 通过 FSQ 的 straight-through estimator 把物理 tracking gradient 传回 encoders；reconstruction、alignment 和 cycle losses 主要更新 encoders 与 motion decoder。

## 它是 on-policy 还是 off-policy

SONIC 使用标准 **on-policy PPO**：每张 GPU 运行 4,096 个并行环境，rollout length 24，训练 5 epochs、4 mini-batches，`γ=0.99`、GAE `λ=0.95`、PPO clip 0.2。最大训练使用 128 GPUs、50k iterations，约 21k GPU hours。

训练采用 asymmetric actor-critic：critic 可以读取仿真的 base linear velocity、完整 body link pose/orientation 和无噪声观测；actor 只读取真机可获得的 noisy proprioceptive history 与 motion command/token。与 BFM-Zero 相同的是 privileged information 只用于 critic；不同的是 SONIC 是显式 tracking reward + PPO，而非 replay-buffer FB learning。

## Kinematic planner 是什么：SONIC 为什么不只会照着预录动作播放

纯 tracker 只能执行已有 reference。为接收速度、方向、姿态高度、风格和 boxing 等在线命令，作者另外训练一个 **generative kinematic motion planner**。

其输入是：

- 当前/历史 pose keyframes；
- 由用户命令生成的目标 keyframe，例如 1 秒后的 root position/heading、目标 pelvis height 或一个 boxing pose；
- style/skill representative clip。

planner 在 latent motion space 中做 masked-token in-betweening，生成连接当前姿态和目标 keyframe 的 0.8–2.4 s 短 motion segment。运行时每约 100 ms 或命令变化时重新规划：

```text
gamepad速度/方向/风格命令
          ↓
root spring model + target keyframe
          ↓
masked-token kinematic planner
          ↓ 0.8–2.4秒short-horizon reference
robot encoder / universal token
          ↓
SONIC tracking policy
```

critically damped spring model 只负责平滑 root position 和 heading 命令，避免从 `+6 m/s` 瞬间反向到 `−6 m/s` 等不现实 command；真正生成全身过渡动作的是 planner。该 planner 是应用层，可以被其他 planner 替换，并不是 PPO tracker 本身。

## 视频、文字、音乐和 VR 到底怎样进入 SONIC

### 视频/文字/音乐

外部 **GEM human-motion model** 接收 RGB video、text 或 music，输出 human motion sequence，再送入 `E_h`：

`video/text/music → GEM → human motion → E_h → token → D_c → action`。

所以 SONIC controller 没有直接看 RGB 或文字。自然语言能力来自 GEM，SONIC 负责把生成的人体运动稳定地落到真实 G1 上。

### VR whole-body teleoperation

PICO headset、手柄和 ankle trackers 产生完整 SMPL pose，通过 `E_h` 转 token。

### 3-point teleoperation

只有 head 与双 wrist 的 SE(3) pose、手指角度、waist height 和 navigation command；`E_m` 处理 sparse upper body，planner 补 lower body。

## 与 VLA 的关系：SONIC 自己并不是那个 VLA

作者用 VR/teleoperation 系统收集 downstream demonstrations，再 fine-tune **GR00T N1.5 VLA**。VLA 读取真实视觉和语言，输出：

`64维 SONIC universal motion token + 14维 hand joints = 78维 action`。

随后 SONIC control decoder 将 64 维 token 与当前本体状态转换成 29 维全身关节目标。因此系统分工是：

```text
GR00T VLA：看环境、理解任务、决定下一段全身motion token和手指动作
SONIC：把motion token变成稳定、自然、物理可执行的全身关节控制
```

这与让 VLA 直接预测 81 维 SMPL pose 相比，在三个任务上的平均成功率从 27% 提高到 68%，平均高 42 个百分点。复杂 soda-can-to-trash-can 任务中是 60% 对 0%。这说明 token 的重要价值不是“压缩得更小”而已，而是限制 VLA 在一个由大规模物理 tracking 学到的可执行动作流形中输出。

## 部署流程

真实 G1 使用最大 42M 模型，全部 inference 运行于 onboard Jetson Orin，TensorRT + CUDA Graph：

- policy inference：50 Hz，单次约 1–2 ms；
- motor command streaming：500 Hz；
- operator input：100 Hz；
- kinematic planner：10 Hz，motion generation 约 12 ms。

真实 policy 输入与训练 actor 的非特权输入一致，不需要外部 mocap 定位来得到当前完整 body state。不同应用通过选择 `E_r、E_h、E_m` 或直接提供 token 切换接口，不需要重训低层 tracker。

## 核心实验结果

### Scaling 是否真的成立

作者分别扩展：

- 数据：4M、10M、22M、100M frames；
- 模型：1.2M、16M、42M 参数；
- 计算：16、32、128 GPUs，约 2k、9k、21k GPU hours。

三条 scaling 轴上 OOD 和 held-out repetition performance 都持续改善。最大模型在 test-content 达到 99.6% success、23.8 mm MPJPE-L，而 1.2M 模型为 98.0%、27.7 mm。绝对 success 提升看上去不大，是因为小模型基线已经很高；更明显的提升体现在 OOD tracking error 与尾部失败动作。

### 和 tracker baseline 比较

统一 MuJoCo evaluation 下，SONIC 在 test-content/test-repetition/PHUMA 达到 98.5%/99.2%/97.2% success；BeyondMimic 为 82.0%/85.4%/73.8%，Any2Track 为 61.6%/69.4%/78.5%。但作者明确承认这些方法使用不同训练数据和 retargeting pipeline，因此结果同时反映数据规模与方法差异，不是严格 data-matched 算法比较。

### 真实 G1 motion tracking

124 条多样 reference motion 中，仿真 124/124 成功，真机 123/124 成功，即 99.2%；真实 MPJPE-L 为 25.7 mm，仿真为 22.3 mm。真机每条 motion 只测一次，因此覆盖面强，但统计重复性仍有限。作者还展示约 11 kg 物体从头顶高度砸向运动中的机器人，策略没有 recovery module 或适配仍保持平衡继续跟踪。

### VLA loco-manipulation

GR00T N1.5 + SONIC token 在 5 类真实任务平均约 75% 成功，包括拿取物体、脚踩垃圾桶踏板、拿饮料罐走到垃圾桶并单脚开盖投放、搬运钻头与箱子。每项为 10–20 trials；它证明统一 token 能支持手脚协同，但规模尚不足以说明开放世界通用 manipulation。

## 关键消融

1. **FSQ vs. VQ-VAE**：FSQ 在 test-content 的 MPJPE-L 优于 VQ-VAE 8.7 mm，且避免 codebook collapse。
2. **Token capacity**：增加 token dimension/levels 都改善结果，其中维度比量化 levels 更重要；默认采用两个 `32维×32 levels` token。
3. **三个 encoder**：robot/human/hybrid encoder 的 success 都超过 99.2%；human encoder 相比 robot encoder 只差约 0.6 mm MPJPE-L。
4. **Alignment losses**：移除 token/cycle consistency 后 cross-encoder divergence 增加约 8 倍，说明共享空间不会仅靠同一个 decoder 自动形成。
5. **VLA action space**：直接预测 SONIC token 明显优于直接预测 SMPL pose，证明 token 是 downstream interface 的实质贡献。
6. **Scaling**：数据、模型、计算三轴实验是整篇最核心消融，而不是只展示最大模型。

## 算法创新是否基本只有离散化

不是，但也不应把每个部件都说成全新算法。

- **FSQ 本身不是 SONIC 发明的。**Finite Scalar Quantization 是已有的离散表示方法；SONIC 的贡献是证明它适合做可由 PPO 端到端塑形、又可供 VLA 预测的人形 motion token。单独把 VQ-VAE 换成 FSQ 只是一项组件选择。
- **更实质的算法设计是 multi-interface latent alignment。**三个 encoder 分别接收 robot、human 和 hybrid command，却用 reconstruction、pairwise token alignment 与 cycle consistency 约束到同一物理可执行空间。
- **它把 representation learning 与 physics RL 联合起来。**token 不是先由纯 AE 在运动数据上学好再交给 tracker，而是同时受到 PPO tracking gradient 和表征 loss 约束。
- **它学到了在线 human-to-robot translation。**`E_h → token → D_r/D_c` 将离线 retargeting pairs 蒸馏为在线神经映射。
- **它把 token 证明为 downstream action interface。**planner、VR、GEM 和 VLA 共用同一底层 decoder；尤其 VLA-token 对 SMPL-action 的对比说明这个接口不仅是压缩表示。
- **系统性 scaling 也是主要贡献。**611 小时动作、42M policy、128 GPUs，以及数据/模型/算力三轴实验，是论文标题中 “Supersizing” 的核心，而非某个单独的新 RL loss。

因此可以概括为：**SONIC 的原创性主要来自“规模化物理 tracking + 多输入统一 token + 可供上层模型复用的控制接口”的组合与验证，而不是提出了一个全新的 RL 算法；FSQ 只是其中重要但非原创的离散化工具。**

## 和 BFM-Zero 的根本区别

| 问题 | BFM-Zero | SONIC |
|---|---|---|
| 基础训练目标 | 无下游 task reward 的 FB successor representation + motion regularization | 显式 reference motion tracking reward |
| RL 算法 | online、off-policy FB-CPR | on-policy PPO |
| `z` 的含义 | reward/task direction：希望长期访问什么状态 | short-horizon motion token：接下来身体怎样运动 |
| `z` 怎么得到 | `B(goal)`、reward projection、motion window，或 latent search | `E_r/E_h/E_m` 编码 motion command 后 FSQ；VLA也可直接预测 |
| 下游接口 | pose、motion、state reward | planner、human motion、VR、视频/文字/音乐生成器、VLA |
| 数据规模 | LAFAN1 40 条数分钟动作 | 611 小时、1亿+帧、317k clips |
| 训练成本 | 约1.92亿环境步、高UTD off-policy | 128 GPUs、21k GPU hours PPO |
| foundation重点 | 用统一 task latent 重组任务 | 用规模化 motion prior 和 token 成为统一低层动作接口 |

两篇都叫 behavior/motion foundation，但 philosophy 不同：BFM-Zero 强调**一个 latent 如何表示多种 reward/goal**；SONIC 强调**一个大规模物理 tracker 如何成为各种高层系统的统一动作基础设施**。

## 局限与我的判断

- 只验证 Unitree G1，尚不能说明 token 或 policy 可直接跨 embodiment；human input compatibility 不等于 robot embodiment generalization。
- 它仍然依赖未来 reference/token，并没有自主理解任务；导航、文字、音乐和 VLA 都依靠额外 planner/model。
- “无需 reward engineering”不完全准确：它避免逐任务 reward，但 tracking reward 本身包含多项人工设计。
- 611 小时完整训练数据并未全部公开，公开 BONES-SEED 为其中 288 小时；外部团队复现最大 scaling result 仍需要非常大的算力与数据。
- planner 使用 keyframe、style clip、root spring 等人工接口设计，并不是一个端到端 perception-to-action 系统。
- 作者明确承认缺乏长期安全与能耗的正式处理；极端不合理 command 或高动态 motion 仍可能失衡。
- VLA 的 5 类任务、10–20 trials 很有说服力但仍是受控任务集；foundation controller 并不自动带来开放世界 manipulation。

**最终判断**：SONIC 比很多只因“一个 latent policy”就自称 foundation model 的论文更有资格使用这个名称，因为它确实展示了大规模数据/模型/计算的系统 scaling、统一接口、OOD motion generalization、大规模真机 tracking 和多类 downstream consumers。但它是 **motion/control foundation**，不是负责视觉语义推理的完整机器人 foundation model。其真正贡献不是发明新的 PPO 控制问题，而是证明「海量 motion tracking + 物理可执行离散 token」可以成为人形机器人上层 planner/VLA 的稳定动作底座。

# BFM 相关工作脉络：它们和 BFM-Zero 到底哪里不同

## 先澄清：BFM 目前不是一个统一算法名

当前文献中的 **Behavior(al) Foundation Model** 更像研究愿景：从大量行为数据或环境交互中得到一个可复用的低层 whole-body policy。满足这个愿景的方法至少有两条明显不同的路线：

1. **task/reward latent 路线**：学习“想优化什么”的统一向量，典型是 Meta Motivo 和 BFM-Zero 的 Forward–Backward representation；
2. **motion tracking foundation 路线**：用大量 reference motion 训练通用 tracker，再通过 mask、token、planner 或感知模块服务不同任务，典型是 Behavior Foundation Model、SONIC 和 ScaleBFM。

因此不能因为论文都使用一个 latent、都能做 motion tracking，就认为方法相同。需要问的是：**latent 表示任务还是动作？新任务是直接编码 reward，还是必须先变成 reference motion/control signal？**

## 按时间排列

| 时间 | 工作 | 核心预训练方式 | 真正的通用接口 | 实物 |
|---|---|---|---|---|
| 2025-04 / ICLR 2025 | [Meta Motivo / Zero-Shot Whole-Body Humanoid Control via BFMs](https://arxiv.org/html/2504.11054) | MuJoCo 中 online off-policy FB-CPR；无标签 AMASS mocap 用 discriminator 作行为正则 | motion、goal、任意 state reward 都投影为 `z` | 否，SMPL 仿真人 |
| 2025-09 | [Behavior Foundation Model for Humanoid Robots](https://arxiv.org/html/2509.13780) | PPO proxy tracker + masked DAgger online distillation + CVAE | root/link/joint goal 的任意 mask；CVAE behavior latent | 是，Unitree G1 |
| 2025-11 / ICLR 2026 | [BFM-Zero](https://arxiv.org/abs/2511.04131) | 将 Meta Motivo 的 FB-CPR 改造为可 sim-to-real 的 G1 policy | motion、goal、reward prompt 的共同 `z`，另支持 latent search | 是，Unitree G1 |
| 2025-11 / Science Robotics 2026 | [SONIC](https://arxiv.org/html/2511.07820v4) | 611 小时 reference motion 上的大规模 PPO tracking + FSQ | robot/human/hybrid motion token，可供 planner/VLA 调用 | 是，Unitree G1 |
| 2026-06 | [Perceptive BFM](https://arxiv.org/html/2606.08059) | terrain-conformal reference 合成 + PPO teacher + DAgger/MSE distillation + PPO fine-tuning | raw motion reference + onboard height map | 是，但真机主要是定性展示 |
| 2026-06 | [ReactiveBFM](https://arxiv.org/html/2606.30362) | diffusion motion planner + 冻结的 ScaleBFM tracker | text/目标位置/本体历史生成在线 reference | 是，Unitree G1 |
| 2026-07 | [Scaling Behavior Foundation Model](https://arxiv.org/html/2607.15163) | 1.02 亿 retargeted frames、64-GPU on-policy PPO、Humanoid Transformer | 多种稀疏/完整 reference control mode | 是，Unitree G1 |

## 1. Meta Motivo：BFM-Zero 的直接前身

Meta Motivo 与 BFM-Zero 不是“相似但独立”的方法，而是同一条算法谱系：

```text
Forward–Backward representation
          +
Conditional Policy Regularization (mocap discriminator)
          ↓
Meta Motivo：在 SMPL/MuJoCo 仿真人上验证
          ↓ 加入G1 embodiment、历史actor、privileged critic、
             domain randomization、安全reward和真机工程
BFM-Zero：真实 Unitree G1 部署
```

两者都学习 `F(s,a,z)^T B(s')`，将 motion、goal 和 reward 映射到同一个 task vector `z`，再由 `π(a|s,z)`执行。Meta Motivo 的核心科研贡献是 FB-CPR 本身；BFM-Zero 的主要新增价值是把这套方法从高维 SMPL 仿真人推向实际 G1，并系统处理 sim-to-real。

关键区别：Meta Motivo 只在 MuJoCo 的 23-actuator SMPL humanoid 上实验，没有真实机器人，也没有处理真机仅凭 IMU/关节历史时的部分可观测问题。它更像 BFM-Zero 的算法原型，而 BFM-Zero 是机器人化、真机化版本。

## 2. Behavior Foundation Model：同名但不是 FB

这篇最容易和 BFM-Zero 混淆。它的完整流程是：

```text
AMASS reference
   ↓
PPO训练 privileged proxy tracker
   ↓ teacher action
当前BFM在仿真中rollout（DAgger）
   ↓
CVAE：prior(real observation + masked goal)
      posterior(privileged state + goal + mask)
      decoder(real observation + latent) → joint target
```

它用统一 control interface 表示 root pose/velocity、link positions 和 joint angles，再对各维做随机 binary mask，于是同一模型可接 full-body reference、VR keypoints、velocity command 等不同低层 control modes。CVAE decoder 故意不直接读取 goal，迫使 latent 携带行为意图。

但它与 BFM-Zero 有三个根本差别：

- 它先用 PPO 训练一个 reference tracker，然后用 DAgger 的 teacher action MSE 与 KL loss 蒸馏 BFM；**不是 reward-free FB learning**。
- 它所谓任意 prompt 实际是对一组预定义低层控制量做任意 mask。论文用文字命令作概念例子，但实际方法没有语言 encoder。
- 未见行为主要通过 residual RL 快速学习，或对已有 latent 插值/外推；**不能像 BFM-Zero 一样把一个新 state reward 直接积分成 `z` 并 zero-shot 执行。**

它的优势是 control mode 很灵活、CVAE latent 具有组合性，而且真实 G1 已验证；代价是训练链条依赖 PPO teacher 和 teacher action，能力仍被 reference dataset/proxy tracker 覆盖范围限定。

## 3. SONIC：把“大 tracker”变成 tokenized motor prior

SONIC 与 BFM-Zero 的共同点是都想让一个低层 policy 服务很多上层任务，但接口完全不同：

- BFM-Zero 输入的是**任务方向** `z`；新 reward 可通过 `B(s)` 投影得到 `z_r`。
- SONIC 输入的是**短时动作意图** token；新任务必须由 planner、GEM、VR 或 VLA 先决定下一段 motion。

因此 BFM-Zero 更接近“通用 reward-conditioned controller”，SONIC 更接近“通用 whole-body motor tokenizer/decoder”。SONIC 的 motion coverage、数据规模、真实动作多样性和上层 VLA 接口明显更强；BFM-Zero 在任务表述的理论统一性和 zero-shot reward optimization 上更强。

## 4. Perceptive BFM：补上 terrain perception，但仍是 reference tracking

Perceptive BFM 指出普通 BFM 默认 reference 与地形物理兼容；平地 backflip reference 放到楼梯上并不会自动知道落脚点。它的四阶段方法是：

1. TCRS 根据 height field、接触与脚部几何，把 flat-ground raw motion 离线改成 terrain-conformal reference；
2. blind teacher 用 PPO 跟踪修改后的 reference；
3. 将 teacher 的有效 PD target 换算到 raw-reference action frame，蒸馏给读取 height map 的 student；
4. student 在 raw reference + height map 下继续 PPO fine-tuning。

部署时只需 raw reference、本体状态和机载 depth-to-height-map，不在线调用 TCRS。真实 G1 展示了楼梯、障碍和高动态动作，但作者明确将硬件结果定位为定性 evidence，量化主要来自仿真。

与 BFM-Zero 相比，它真正解决的是**同一动作如何因地形而改变落脚和姿态**，而不是如何用 reward prompt 产生新任务。它有外部感知，但假设静态、刚性、可观测 height field，且上肢 reference 仍可能撞障碍；还不是视觉语义导航或通用 loco-manipulation。

## 5. ScaleBFM：研究怎样把 tracking-based BFM 真正扩展大

ScaleBFM 延续 Behavior Foundation Model/SONIC 所在的 tracker 路线，核心结论有三点：

- 在 PPO 中，“训练数据量”不只是 reference clips 数目，更是每次更新收集的 on-policy rollout 数量；reference motion 主要控制数据**多样性**。
- 使用 1.02 亿个 50 Hz human-motion frames，经过两阶段 skeleton alignment + frame-wise IK retarget 到 G1，并用 64 GPUs、longer rollout 扩大 on-policy interaction。
- 提出 Humanoid Transformer，以历史 proprio/action token 和未来 goal tokens 学 structured behavior latent；action 仍是 joint position targets。

它支持 global 和 local control，但要谨慎解释 local mode：作者把 reference 的当前 root position 重新锚定为机器人当前 root position，相当于假设“平移目标已经到达”，只继续追踪朝向与相对全身动作；global mode 则用 HTC Vive tracker 提供 root localization。因此它不是不需要定位的自主 goal-reaching policy。

与 BFM-Zero 相比，ScaleBFM 的重点是 **reference-tracking fidelity、数据/模型/rollout scaling 和多 control modes**；它没有 FB 的 reward projection，也不能把任意 reward 直接变成策略。它在动作覆盖和 tracking 精度上更可扩展，但仍需上层系统先给出要执行的 reference/control signal。

## 6. ReactiveBFM：给 ScaleBFM 加一个真正闭环的 motion planner

ReactiveBFM 不是重新预训练一种 BFM，而是将冻结的 ScaleBFM 当低层控制器，在其上加 auto-regressive diffusion motion planner：

```text
text / 3D target / executed proprioceptive history
                    ↓
     closed-loop diffusion motion planner
                    ↓ online reference chunks
              frozen ScaleBFM
                    ↓
             joint position targets
```

planner 通过 scheduled prefix sampling 看自己的历史预测而非永远看 ground truth，从而减轻 autoregressive exposure bias；异步重规划和 overlapping trajectory ensemble 解决 planner 与 50 Hz controller 的时延差异。真实动态目标追踪依赖两个 HTC Vive trackers 获得机器人与目标的全局位置，并将动态追踪拆成不断重置坐标系的静态 reaching 子任务。

它相对 BFM-Zero 的优势是能接受 text/target 并持续基于真实执行状态重规划，长时动作不会只是开环播放；但它是明确的 planner–controller 两层系统，训练数据经过动作生成、retargeting 和物理筛选，且没有 RGB/tactile 输入。论文所谓 zero-shot dynamic reaching 是“未用动态目标训练”，并不等于无需外部定位或能在未知环境自主导航。

## 横向比较：BFM-Zero 的独特位置

| 维度 | BFM-Zero | BFM/ScaleBFM | SONIC | Perceptive/Reactive BFM |
|---|---|---|---|---|
| 预训练在学什么 | 多种长期任务的 successor/task representation | 各种 reference/control mask 下的通用 tracking policy | 可执行 motion token 与大型 tracker | 对 tracking backbone 增加地形适应或闭环 planner |
| latent/token 物理含义 | “长期希望访问哪些状态”的 reward direction | 行为/控制规格的连续 latent | 下一小段身体运动 | reference correction 或 planner output |
| 新 reward 能否直接接入 | **可以**，投影为 `z_r` | 不可以，需化为 control signal 或再训练 | 不可以，需 planner/VLA 产 token | 不可以，需 reference/target/text planner |
| 是否需要 reference tracker teacher | 不需要 action-labeled teacher；用 mocap discriminator 正则 | BFM 需要；ScaleBFM直接 PPO tracking | 直接 PPO tracking | 需要 tracking backbone/teacher |
| 感知 | 仅本体；prompt 构造可能需要外部/特权状态 | 主要本体 + reference | 本体 + token；视觉在外部 VLA | Perceptive 有 height map；Reactive 仍无视觉环境输入 |
| 真机 | G1 | G1 | G1 | G1 |
| 最强项 | 统一 motion/goal/reward prompt，zero-shot reward optimization | tracking/control-mode覆盖与规模化 | 海量 motion、多输入 token、VLA action interface | 地形适应或实时闭环生成 |
| 核心限制 | 数据小、无环境感知；reward/goal prompt 未必能在真机无特权地构造 | “通用”通常仍限于 reference/control interface | 需要上层决定 motion，不理解任务 | 模块多、假设多，仍未形成端到端感知—决策—控制 |

## 我的判断

1. **和 BFM-Zero 算法上最近的是 Meta Motivo，而不是 SONIC 或原名为 BFM 的论文。**后几篇多数属于 tracking foundation 路线。
2. **BFM-Zero 的独特贡献不是动作更多，而是任务表示更一般。**它首次在真实 G1 上展示 motion、goal、reward 可进入同一个 FB latent，并允许不更新 policy 的 zero-shot/few-shot latent inference。
3. **BFM-Zero 并没有在所有意义上更“foundation”。**它的 mocap 规模与真实动作覆盖远小于 SONIC/ScaleBFM；也没有 Perceptive BFM 的地形输入或 ReactiveBFM 的实时高层 planner。
4. **tracking-based BFM 的工程成熟度更强，但任务通用性容易被夸大。**它们通常要求外部模块把任务变成 reference、keypoints 或 token；能跟踪很多动作不等于能理解任意 reward 或自主完成未知环境任务。
5. **这些路线可能最终合流。**最合理的未来系统可能是：视觉/语言/world model 做高层任务理解与闭环规划，SONIC/ScaleBFM 提供大规模 motor prior，FB-style latent 提供 reward/task adaptation，感知型 residual 则处理地形和接触。当前没有一篇把这几部分全部解决。

# LATENT：Learning Athletic Humanoid Tennis Skills from Imperfect Human Motion Data

- 时间：2026-03，arXiv:2603.12686 v1；截至当前论文页面未标注正式会议/期刊录用
- 作者/单位：清华、北大、Galbot、上海启智研究院、上海AI Lab
- 机器人：29-DoF Unitree G1，右手替换成通过3D打印连接件固定的标准网球拍
- 仿真器：MuJoCo JAX / MuJoCo Playground，8张GPU并行训练
- 算法：PPO + motion tracking + DAgger式online distillation + conditional VAE latent action space
- 项目页：https://zzk273.github.io/LATENT/
- 论文：https://arxiv.org/abs/2603.12686
- 代码：https://github.com/GalaxyGeneralRobotics/LATENT

## 一句话总结

LATENT不是从一段完整网球比赛动作直接模仿，而是先从5小时、不完整且手腕不精确的人类网球动作片段中学习forehand、backhand、shuffle、crossover等primitive skills，再把这些技能蒸馏成一个可被高层RL调用的latent action space。高层PPO负责根据机器人和球的状态组合这些技能，同时绕过latent单独修正持拍手腕，最终在外部动捕覆盖的场地中让G1完成真人来球回击和多拍回合。

## 任务到底是什么

任务不是完整的网球比赛，而是tennis return：

- 每个episode开始时，G1以ready pose随机放在球场一侧。
- 仿真中需要连续回击8个来球，每2秒发射一个球。
- 每个球的初始位置和速度随机，机器人需要跑到合适位置，用正手或反手把球击回指定落点。
- 任务同时要求到球、击中球、过网、落在目标附近、动作自然、关节和力矩安全，并保持平衡不摔倒。
- 机器人不负责发球、比分与战术决策，也没有学习如何与真实对手进行完整的双向博弈。

因此它本质上是一个高动态、全身协调的球拍拦截与目标落点控制任务，而不是完整“会打网球”的通用系统。

## 为什么需要latent action space

如果让PPO从29个关节动作中从零探索：

- 很难同时学会移动、保持平衡、挥拍和精确击球；
- 即使偶然击中球，也容易得到僵硬或不自然的动作；
- 数毫秒的球拍接触对手腕角度和速度非常敏感，稀疏成功奖励难以指导整个身体。

作者因此先从人类数据中学习一个低维动作先验。高层policy不直接随意控制所有身体关节，而是在由人类primitive skills构成的latent manifold中选择和组合动作。这个latent表示的是**可执行动作/技能空间**，不是RMA中用于编码未知动力学参数的environment latent，也不是语言模型的token latent。

## “Imperfect Human Motion Data”是什么

作者邀请5名业余网球运动员，在一个约3 m × 5 m的光学动捕区域内采集约5小时数据。只采集局部技能片段，包括：

- forehand stroke；
- backhand stroke；
- lateral shuffle；
- crossover step；
- 其他网球相关基础移动和挥拍动作。

数据不包含完整比赛或连续多拍轨迹，不做人工编辑和动作标签，之后通过LocoMuJoCo重定向到Unitree G1。

这里的“imperfect”有两层含义：

1. Imprecise：持拍手腕的细微角度很难准确捕捉，人到机器人retargeting又会进一步放大误差，直接照抄往往打不中球。
2. Incomplete：这些片段只告诉机器人“像人一样正手、反手、横移”，没有告诉它面对某个来球时何时移动、选哪个动作、怎样把球回到指定位置。

需要注意：它并不是随便使用低质量视频。数据仍来自受控的光学动捕，只是采集者是业余球员、场地小、只包含primitive fragments，而且关键手腕运动不够精确。

## 三阶段训练流程

### 第一阶段：Motion tracker pre-training

首先用Any2Track风格的PPO motion tracker把retarget后的人类动作变成G1能够在动力学仿真中稳定执行的技能。

Tracker输入：

- 当前机器人状态 $s_t$：base angular velocity、projected gravity、关节位置、关节速度、上一帧动作；
- 下一帧参考动作状态 $\tilde{s}_{t+1}$。

Tracker输出：

- 除持拍右腕外的body joint position targets；
- position target再经过PD controller得到关节力矩。

训练目标是让下一时刻机器人状态接近reference motion，同时维持平衡与安全。

#### 为什么故意不让tracker控制右腕

人类动捕和retarget得到的右腕动作最不可信，但击球最依赖右腕精度。作者没有强迫整个tracker去拟合这个错误信号，而是：

- 从tracker动作中移除持拍右腕的控制信号；
- 训练时给右腕加入随机扰动；
- 强迫身体和下肢策略在未知右腕动作下仍然保持稳定。

这样为后面高层policy直接修正右腕留出接口。这就是所谓correctable latent action space的基础。

### 第二阶段：Online distillation得到latent action space

预训练tracker虽然会跟踪动作，但运行时仍然需要下一帧完整reference。作者随后把它通过DAgger式online distillation蒸馏成encoder-decoder结构。

**这一阶段的目的**：不是再把一段reference录成可检索的动作库，而是学一个连续的“低维技能控制接口”。只要给decoder一个latent code $z$ 和当前机器人状态 $s_t$，它就能输出可执行的全身关节目标，不必再直接输入下一帧reference。这里的online是指student在仿真中自行rollout、teacher在student到达的状态上给动作标签；不是真机部署时在线更新参数。

虽然外形像VAE/AE，训练目标并非重建输入的reference，而是让decoder输出的**动作**匹配teacher tracker，同时用KL组织latent分布；称作“带变分瓶颈的动作蒸馏”比普通autoencoder更准确。

Posterior encoder：

$$
\mathcal E(z_t^q\mid s_t,\tilde{s}_{t+1})
=\mathcal N(\mu^e,\sigma^e),
$$

输入当前状态和reference，输出Gaussian posterior的均值与标准差，从中采样$z_t^q$以编码“这一帧reference对应的技能/动作”。

Decoder：

$$
\mathcal D(a_t^{body}\mid s_t,z_t^q),
$$

根据当前状态和latent code输出除右腕外的joint position targets；同一个$z$在不同姿态/速度下会解码成不同的具体关节动作。它不是把$z$查表成一段固定的预录动作。

蒸馏loss由两部分组成：

$$
\mathcal L=\lambda_1\mathcal L_{action}+\lambda_2\mathcal L_{KL}.
$$

- $\mathcal L_{action}$：student action拟合teacher tracker action；数据来自student rollout后的aggregate buffer，因此是DAgger式online distillation，不只是对固定teacher dataset做behavior cloning。
- $\mathcal L_{KL}$：让posterior靠近一个依赖当前robot state的conditional prior $\mathcal P(z_t^p\mid s_t)$。

Conditional prior很重要：横移、挥拍等不同状态下合理的动作分布不同，不适合全部压到同一个固定标准高斯。prior输出每个状态下latent的均值 $\mu_t^p$ 和标准差 $\sigma_t^p$，后面也被用于限制高层policy的探索范围。

#### 为什么还需要一个没有reference的conditional prior？

蒸馏时的posterior encoder $q(z\mid s_t,\tilde{s}_{t+1})$ **看得到reference**，所以能够把指定动作编码成$z$；但真机打球时没有人持续提供下一帧人类动作，不能靠它选择技能。于是另训一个只看当前机器人状态的conditional prior $p(z\mid s_t)$：

$$
p(z\mid s_t)=\mathcal N(\mu_t^p,\sigma_t^p),\qquad
\mathcal L_{KL}=D_{KL}\big(q(z\mid s_t,\tilde{s}_{t+1})\,\|\,p(z\mid s_t)\big).
$$

这不是让有/无reference两个encoder的输出做MSE并强制相同。两者输出的是**分布**，KL同时约束均值和方差。Prior学的是“处于当前状态时，latent动作大致落在哪些合理、可执行的范围”，而不是“当前一定应该打正手”的唯一答案。选择哪一种具体动作，要等第三阶段看见球状态的高层policy来决定。

特别地，同一个$s_t$可以对应向左横移、向右横移或挥拍等多个reference；只有reference的posterior可以分别编码它们，只看$s_t$的prior不可能无中生有地知道意图。一个对角Gaussian只能近似覆盖这些可能的latent，未必能精确表达相互分离的多模态技能；这是需要注意的方法假设。

这一阶段训练的主体是posterior encoder、conditional prior和action decoder。蒸馏完成后，posterior encoder和teacher tracker不参与第三阶段的任务决策；实际要用的是不依赖reference的prior与decoder。

### 第三阶段：High-level tennis policy

**这一阶段的目的**：第二阶段只学会“有哪些合理技能以及如何执行”，还不知道来球在哪、何时跑动、何时挥拍、该用正手还是反手。第三阶段用PPO训练高层planner，根据球和机器人的状态选择、组合技能并修正关键末端，使球落到目标区域。它不再使用人体reference。

方法上是复用预先学好的latent action model，让高层planner在其中学习任务动作。通常可把prior和decoder理解成第三阶段的预训练、固定的动作接口，posterior encoder则不再使用；不过论文v1没有明确列出第三阶段到底冻结了哪些参数，也未公开完整高层训练代码，因此**不能把“prior和decoder一定被冻结”当成已核实的实现细节**。

高层policy输入：

- robot proprioceptive state $s_t$；
- 机器人root的全局位姿信息 $g_t$；
- 网球的位置和速度 $b_t$。

高层policy输出两部分：

$$
a_t^{planner}=[a_t^{latent},a_t^{correct}].
$$

- $a_t^{latent}$：PPO输出的**归一化latent残差指令**，用于在当前prior中心附近选择/组合全身primitive skills；它不是最终交给decoder的$z_t$，也没有直接的速度、力矩或关节角等物理单位。
- $a_t^{correct}$：绕过latent decoder，直接提供持拍右腕的关节位置控制目标，再经PD控制器转成实际关节力矩；它不是原始电机扭矩或电流指令。

Decoder再把latent action解码成其余身体关节目标，与右腕修正动作拼接成完整PD target。高层planner和低层decoder都运行在50 Hz，MuJoCo物理仿真运行在2000 Hz，以更准确模拟球拍—球和球—地面的高速接触。

## Latent Action Barrier（LAB）

若高层PPO能无约束地输出任意latent，它会找到“能打中球但不像人”的漏洞，例如在不同移动技能之间快速来回跳变，产生抖动和大力矩动作。

LATENT不是只在prior均值附近加一个固定Euclidean residual，而是利用conditional prior给出的逐维标准差形成state-dependent barrier：

$$
z_t=\mu_t^p+\lambda\sigma_t^p\odot\tanh(a_t^{latent}),
$$

$$
a_t^{full}=
[\mathcal D(s_t,z_t),a_t^{correct}].
$$

这里要分清四个变量：$p(z\mid s_t)$给出合理latent的**分布**；$\mu_t^p$是其中心；$a_t^{latent}$是高层policy输出的原始残差动作；经$\tanh$、$\sigma_t^p$和$\lambda$缩放以后得到的$z_t$才是decoder真正接收的latent code。可改写为$z_t=\mu_t^p+\delta z_t$，其中$\delta z_t=\lambda\sigma_t^p\odot\tanh(a_t^{latent})$。作者用$a$表示它是RL policy的action，用$z$表示变换后的latent，并非二者有两套不同的物理语义。

直观上：

- prior均值表示当前状态下最典型的人类技能；
- prior方差大的latent方向允许policy探索得更远；
- prior方差小的方向被更严格限制；
- $\tanh$ 和尺度 $\lambda$ 防止policy跑出动作数据支持范围。

作者把它解释为Mahalanobis-distance意义下的latent barrier。它的作用不是让policy永远复制reference，而是在“完成回球任务”和“保持人类动作先验”之间形成一个随状态变化的trust region。

LAB也不是传统控制理论中的control barrier function；这里的“barrier”具体是对高层latent residual做有界、state-dependent的变换。并且$\sigma_t^p$只是学习到的latent分布尺度，不能未经验证就把它解释为严格的安全边界。

## 与四足《Learning Multiple Gaits within Latent Space》的关系

两篇的共同思路是：**先让一个有额外动作/步态信息的网络塑造latent空间，再让不直接看到这些信息的策略通过latent控制机器人**。这样高层不必在原始高维关节空间盲目探索。它们都不是一个离散的“动作库”，而是学习到的连续latent接口。

但不能把两篇的网络逐个对应：

| 问题 | 四足Multiple Gaits笔记 | LATENT |
|---|---|---|
| 额外动作信息 | 人为设计的步态参数，如频率、身体高度等 | 人体动捕得到的下一帧reference motion |
| latent来源 | Gait Encoder编码步态参数，Gait Generator用自身状态/命令对齐该空间 | Posterior encoder编码$(s,reference)$；conditional prior估计仅给定$s$的合理$z$分布 |
| latent如何变成动作 | latent作为四足locomotion policy的条件/内部表征 | 显式的decoder $\mathcal D(s,z)$输出身体关节位置目标 |
| 下游选择 | 目标是找到适应地形的步态表示 | 高层PPO看球和全局状态，输出latent residual以及右腕直接动作 |
| 额外约束 | 通过步态latent对齐与训练设计保持可控 | LAB用prior的$\mu,\sigma$限制latent residual的范围 |

最容易记的区别：四足工作偏向**让机器人自己选择合适步态**；LATENT偏向**在预学的人类运动先验中组合移动/挥拍技能，同时允许关键手腕脱离先验做精确任务修正**。另外，LATENT的prior不是“无reference版本的posterior给出同一个唯一$z$”，而是给高层policy提供一个状态相关的可选动作范围。

## Reward设计

Task reward主要包括：

- Approach to ball：接近来球；
- Hit success：成功击中球，权重200，是最大的正奖励；
- Ball landing：落点接近目标。

Regularization包括：

- high-level/latent action幅度；
- correction action及其变化率；
- 全身和下肢action rate；
- torque、关节加速度/平滑性；
- racket acceleration和wrist torque；
- joint position/velocity limits；
- self-collision；
- 过网高度、球速、球拍速度；
- pelvis朝向球场前方。

Termination penalty包括摔倒、漏球、球撞网、出界和stroke style violation。

这不是仅靠一个稀疏“是否得分”奖励学出来的；奖励设计包含了较多任务阶段、安全和运动风格约束，人类动作latent只是减少了搜索空间。

## Sim-to-real

### Dynamics randomization

机器人侧随机化：

- feet/joint friction；
- armature；
- body mass和CoM；
- racket mass和CoM。

网球侧随机化：

- ball mass；
- ball-ground restitution；
- tangential damping；
- quadratic air-drag coefficient。

空气阻力使用简化模型：

$$
f_{air}=-k m v\|v\|.
$$

作者没有精确辨识真实球拍接触和空气动力学，而是通过宽范围随机化覆盖误差。

### Observation corruption

训练中对机器人和球的观测加入uniform noise、整帧dropout和latency。球速由球位置有限差分得到，对位置噪声非常敏感，所以仿真和真机都使用4帧滑动窗口的平均速度。

## 真机的信息从哪里来

真机并不是用G1机载相机看球。机器人global 6D root pose和球状态来自场外光学动捕：

- 机器人base贴多个反光marker并作为rigid body估计6D pose；
- 网球包覆反光材料并作为passive marker追踪；
- 官方代码README披露使用50多台2048×2048、120 Hz动捕相机；
- 动捕覆盖约19 m × 15 m；
- 场地、动捕和灯光等由第三方服务商提供，约租用3周，成本约35万元人民币/5万美元。

因此这篇的强项是whole-body dynamic control和高速球拍交互，不是active vision或自主球状态估计。它证明policy能在真实机器人动力学与真实球接触下工作，但没有解决只靠机器人自身传感器在普通球场部署的问题。

## 仿真实验

平台为29-DoF Unitree G1。作者在forehand、backhand、forecourt和backcourt四组条件上总计评估10,000次。

指标：

- SR：球落到目标点2.5 m以内视为成功；
- DE：实际落点到目标点的平均距离；
- Smoothness：平均关节加速度；
- Torque：平均关节力矩。

主要对比：vanilla PPO、MotionVAE、AMP、ASE和PULSE。

- PPO从零学习和MotionVAE在作者实现中没有收敛。
- PULSE是最强latent baseline，但缺少右腕correction，且latent的构造/采样方式不同。
- LATENT的success rate分别为forehand 96.52%、backhand 82.10%、forecourt 86.35%、backcourt 89.80%；对应PULSE为71.85%、63.38%、74.52%、72.94%。
- LATENT的落点误差约1.32–1.89 m，同时smoothness和torque也优于AMP、ASE和PULSE。

基线结果说明动作先验确实重要，但需要谨慎解释：PPO/MotionVAE失败只说明在作者的同一训练预算和实现下难以收敛，并不等价于这些范式原则上无法完成任务。

## 消融实验

### 去掉wrist correction

把右腕也交给motion tracker和latent decoder，高层policy不能独立修正。四种设置下成功率分别从96.52/82.10/86.35/89.80%下降到82.36/68.94/74.21/79.05%，落点误差明显增大。

结论：人类motion prior可以提供全身自然动作，但高精度球拍接触仍需要任务policy对关键末端执行器做direct residual correction。

### 去掉LAB

不限制residual latent的范围后，success rate有所下降，更明显的是动作smoothness和torque变差。说明policy会利用latent decoder的数据外区域得到能完成任务但抖动、耗力的动作。

结论：LAB的主要价值不只是提高hit rate，而是防止高层RL破坏motion prior，使latent采样保持在当前状态下合理的技能分布附近。

## 真机实验

作者进行20组连续真人—机器人回合测试，来球初始位置和速度变化，并统计正手/反手以及前场/后场表现。

完整方法的success rate：

- forehand 90.90%；
- backhand 77.78%；
- forecourt 88.89%；
- backcourt 81.82%。

但是需要注意，真机成功标准从仿真的“落在目标点2.5 m以内”放宽成“落在对方整个球场边界内”。真机DE仍约3.15–3.89 m，所以高回球率不等于精确落点控制，仿真和真机SR也不能直接横向比较。

真机消融显示球的DR和observation noise非常关键：去掉ball dynamics randomization后成功率只约14–29%；去掉ball observation noise后不同类别为0–50%。这支持了“宽随机化可替代精确球动力学辨识”的工程结论，但真机没有与其他完整方法比较，主要对比的是自身消融。

此外，仿真中的robot-robot self-play在50场随机比赛中最多达到25拍连续回合；这属于额外仿真实验，不应与真人真机多拍表现混为一谈。

## 这篇论文真正的创新点

1. 用小场地、片段式、业余运动员的人体动捕代替昂贵的完整网球比赛数据，再通过RL完成任务层面的技能组合。
2. Correctable latent action space：全身自然运动来自motion prior，但把数据最不可靠、任务精度要求最高的右腕留给高层policy直接修正。
3. Latent Action Barrier：利用state-conditioned prior的均值和方差限制高层latent residual，在任务成功与动作自然性之间做自适应约束。
4. 将高动态全身移动、球拍接触和球落点控制zero-shot迁移到真实G1，并展示真人多拍回合。

它不是第一次使用latent action space，也不是第一次做humanoid sports。基础路线明显继承PULSE/R2S2一类“tracker -> online distillation -> high-level latent policy”的层级框架。真正新增的是如何处理**不完整且关键末端不精确的motion prior**：允许局部correct，同时对其余latent exploration加barrier。

## 局限与需要谨慎看待的点

- 强依赖外部动捕获取机器人全局位姿和球状态，不是视觉驱动，也不是仅靠onboard sensing的自主系统。
- 真机基础设施很重：50多台相机、19 m × 15 m场地和约5万美元的三周租赁成本，限制了可复制性与普通场地部署。
- 只做return task，没有发球、规则、战术、对手意图预测和完整比赛训练；作者自己也承认随机来球设定与真实双人比赛仍有差距。
- 人类数据虽称“imperfect”，但仍是5小时受控光学动捕和retargeted motion，并非无需高质量采集设备。
- 20组真机测试规模较小，分类到正/反手、前/后场后每组样本更少；真机没有与PULSE等完整baseline比较。
- 真机成功标准比仿真宽松，且3–4 m落点误差说明精确控制仍有限。
- correction只针对右腕，默认其余身体motion prior足够可靠；更广泛的数据错误、不同球拍/机器人或损伤情况下是否仍有效没有验证。
- 高层输入包含oracle式global root pose和完整ball state，仿真结果不能被理解为已经解决球感知、轨迹预测和active perception。
- 论文当前是2026年3月的arXiv v1；正式同行评审状态未标明。

## 开源情况

官方GitHub已经公开MuJoCo/JAX motion tracking框架和少量retargeted tennis motion data。README中的TODO仍列出完整数据、预训练tracker、DAgger distillation、latent model、高层网球policy、sim-to-real设计和更多checkpoint尚待发布。因此目前是**部分开源**，还不能仅凭仓库完整复现论文整条pipeline。

## 对我们做人形RL的启发

1. 对于高动态交互，不必执着于采集完整任务演示；可收集容易获得的primitive fragments，再让task RL学习时序选择和组合。
2. Motion prior不应被视为不可修改的硬约束。对数据误差最大、任务精度最高的末端执行器保留direct residual/correction接口，比要求一个latent同时负责自然性和毫米级接触更合理。
3. 用conditional prior的variance限制latent residual，比固定范围clip更有意义：它根据当前状态下数据分布的宽窄决定允许探索多少。
4. 高速球类sim-to-real不仅是机器人动力学问题，交互物体的restitution、drag、接触和观测延迟往往才是主要gap。
5. 真机演示很强并不等于感知问题已经解决。评价一篇humanoid sport论文时，需要分别检查motion generation、whole-body control、object perception、global localization和game strategy分别由谁完成。

### 最终判断

LATENT是一篇很强的**层级式人形运动控制与技能组合**工作。它最值得学习的不是“latent”这个名字，而是对数据缺陷的结构化处理：用motion prior负责自然且稳定的全身运动，用direct wrist correction补偿关键末端误差，再用state-dependent LAB防止RL破坏动作流形。其真机多拍回球证明了高动态控制和sim-to-real的工程完成度，但外部大规模动捕承担了全部球感知与全局定位，因此它距离可在普通球场自主打网球仍有明显差距。
