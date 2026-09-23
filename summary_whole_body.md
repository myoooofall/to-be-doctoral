### MLM: Learning Multi-task Loco-Manipulation Whole-Body Control for Quadruped Robot with Arm：  
go2+umi 训练的时候 可以训了一个跟踪轨迹的全身运动控制器，直接输出18个电机 这个轨迹 可以通过遥操作，也可以通过umi 那套的diffusion 来生成。umi上是有鱼眼相机的 感知+自动生成轨迹，这样相当于policy可以做多任务，因为 本质上 不同任务就是不同轨迹。

### Manipulate-to-Navigate: Reinforcement Learning with Visual Affordances and Manipulability Priors

在机器人上安装了一个相机，首先通过相机 机械臂 机器人的位置 这样计算出图像的每一个区域的可操作度（排除奇异点）这样就可以计算reward了 然后 rl（这里不是ppo 离散的选点）去选出一个 像素点来 再通过外参转换到世界坐标  实际上他不能算一篇全身控制的文章 他这里压根没有讲 底层怎么控制 就是 选一个点 然后用spot 自己的算法 去 让机械臂先到达这个点 然后机械臂不动 底盘开始移动
实现一直抵着门 然后身体走进房间的效果  

这篇意义不大 有点取巧了 也许在导航里有点用 胳膊抵一个东西 比如障碍物 身体走过去 

### Interactive Navigation for Legged Manipulators with Learned Arm-Pushing Controller
推开障碍物导航 但是 这篇 默认有全局地图 动作捕捉 和可以推的物体（默认机器人知道） 高层规划器先a*规划一下路 一堆公式 规划出什么时候要推>绕行 然后再规划一个 下一时刻机械臂关节位置 然后底层wbc 根据位置输入 得到 18个电机的 期望位置

### Humanoid Whole-Body / Loco-Manipulation Controller 相关工作
这一组是WholeBodyVLA related work里点名批评/对比的几篇  
它们大多不是端到端VLA 而是人形机器人的whole-body controller / teleoperation controller / skill prior  
共同特点是：多数仍然采用velocity-tracking interface 也就是让低层policy跟踪base速度、身体高度、手臂pose/joint等命令  
这种接口做巡航和遥操作很方便 但对loco-manipulation里的“走到哪里停、朝向是否准、刹车是否稳、拿东西时负载耦合是否稳定”监督不够直接  

#### HOMIE: Humanoid Loco-Manipulation with Isomorphic Exoskeleton Cockpit
2025 RSS  
HOMIE = Humanoid Loco-Manipulation with Isomorphic Exoskeleton Cockpit 作者自命名  
任务是做人形机器人全身遥操作：操作者用低成本外骨骼手臂 + motion-sensing gloves + pedal 控制Unitree G1 / Fourier GR-1  
能做walking、squatting、抓瓶子、pick-and-place、打开烤箱、搬箱子、双机器人handover等真实遥操作任务  

方法核心不是自动规划 而是“可靠低层身体控制 + 同构外骨骼采集上身命令”  
低层RL policy负责让机器人走路、下蹲到指定高度，同时适应任意变化的上身姿态  
上肢/手通过外骨骼和手套直接给目标关节或手部命令，避免复杂IK retargeting误差  
训练技巧包括 upper-body pose curriculum、height-tracking reward、symmetry utilization  
作者强调MoCap-free，不依赖人类动作捕捉数据做motion prior  

仿真器：Isaac Gym训练低层policy，训练后也转到GRUtopia场景里展示更复杂任务  
真实机器人：Unitree G1 和 Fourier GR-1  
输入输出要非常明确：HOMIE训练的是下半身locomotion policy `π_loco`，不是全身RL策略  
command `C_t = [v_x,t, omega_yaw,t, h_t]`，分别是前后速度、yaw转向速度、torso目标高度  
observation `O_t = [C_t, omega_t, g_t, q_t, qdot_t, a_{t-1}]`  
其中 `omega_t` 是base角速度，`g_t` 是重力投影，`q_t/qdot_t` 是机器人关节位置/速度，`a_{t-1}` 是上一时刻下身动作  
action `a_t` 是 lower-body joint position targets，也就是下半身电机的目标关节位置  
这些目标关节位置再通过PD控制器变成电机力矩  
上半身不是RL policy输出的：`q_upper` 由同构外骨骼直接映射到机器人上身关节目标，手指由motion-sensing gloves映射  
所以真实运行时是：操作者脚踏板给 `C_t`，外骨骼给 `q_upper`，`π_loco` 只根据 `C_t` 和本体状态输出下半身关节目标来稳住走路/下蹲  
是否带感知：低层`π_loco`本身不吃视觉；操作者通过FPV画面看环境并遥操作  
论文后面也用HOMIE采集的数据训练`π_auto` visuomotor imitation policy  
这时机器人把D455/D435相机图像和joint信息发给主机，`π_auto` 输出 `C_t` 和 `q_upper`，再由 `π_loco` 输出下身 `a_t`  
所以自主版本的视觉策略并不直接输出所有电机，它输出原本由人提供的脚踏板命令和上身关节命令  
局限：更像高质量teleoperation/data collection cockpit，不是自主视觉任务策略；WholeBodyVLA批评它的上肢更多是PD/遥操作命令稳定化，任务层自主性和closed-loop VLA能力有限  
链接：https://homietele.github.io/  

#### ALMI: Adversarial Locomotion and Motion Imitation for Humanoid Policy Learning
2025 NeurIPS  
ALMI = Adversarial Locomotion and Motion Imitation 作者自命名  
任务是人形机器人whole-body motion imitation：下半身保持速度命令下的稳定行走，上半身跟踪多样化动作  
它可以扩展到teleoperation式loco-manipulation，但核心仍然是motion tracking / imitation，不是物体交互任务本身  

方法核心是把上身和下身角色分开，用adversarial / iterative learning让二者互相适应  
lower-body policy负责robust locomotion，跟踪velocity commands  
upper-body policy负责跟踪各种上身motion，同时要在机器人走路时仍然可跟踪  
这样避免传统全身模仿时上身动作把下身稳定性拖垮  

仿真器：Isaac Gym训练policy；另用MuJoCo生成/发布大规模whole-body motion control dataset ALMI-X；论文也在真实Unitree H1上验证  
时间：arXiv 2025-04，NeurIPS 2025  
策略输入/输出：这篇我还需要再逐公式确认；目前只按方法定位记录为motion imitation / locomotion controller  
可以确定的是它不属于视觉VLA，输入是proprioception + velocity command + reference/upper-body motion相关信息，输出是关节控制动作，经PD执行  
是否带感知：不带视觉/语言感知，主要是proprioception + command/reference  
上层命令来源：来自motion dataset或teleoperation/retargeted motion，不是自主planner根据图像决定任务  
能做什么：真实H1上做鲁棒行走 + 上身动作跟踪，适合作为teleop/motion imitation底层  
局限：仍然是velocity-command locomotion + motion imitation范式，物体、接触力、任务成功条件不是主训练目标  
链接：https://arxiv.org/abs/2504.14305  

#### AMO: Adaptive Motion Optimization for Hyper-Dexterous Humanoid Whole-Body Control
2025 RSS  
AMO = Adaptive Motion Optimization 作者自命名  
任务是让Unitree G1做人形hyper-dexterous whole-body control，扩大可达工作空间  
典型能力包括弯腰、下蹲、从地上拿物体、双臂/躯干大范围协调，以及通过VR遥操作或上层IL策略执行任务  

方法是RL + trajectory optimization混合  
作者先构造hybrid AMO dataset：上身动作来自AMASS motion / random torso command / VR目标等，再用trajectory optimization生成动态可行的身体参考  
其中trajectory optimization用类似多接触约束和wrench cone约束，保证下半身/身体姿态在动力学上可行  
再训练AMO controller，使它面对OOD command也能实时调整身体姿态和下肢支撑  

仿真器：论文项目页展示MuJoCo仿真表现；训练/部署体系是sim-to-real RL + optimization，真实部署在29-DoF Unitree G1  
时间：arXiv 2025-05，RSS 2025  
AMO的接口要分清楚：它不是一个直接从图像到动作的policy，也不是只输出腿关节的普通locomotion policy  

Teleoperation setting里，goal-conditioned policy形式写成 `π' : G x S -> A`  
goal `g = [p_head, p_left, p_right, v]`  
`p_head, p_left, p_right` 是操作者VR/teleop系统给的头和双手keypoint pose  
`v = [v_x, v_y, v_yaw]` 是base速度命令  
observation `s = [img_left, img_right, s_proprio]`，也就是可以包含双目图像和本体状态  
action `a = [q_upper, q_lower]`，即上半身和下半身关节角命令  

实际系统是分层的：  
upper policy `π'_upper(p_head, p_left, p_right) -> [q_upper, g']`  
它把头/双手pose通过multi-target IK / upper policy变成上半身关节命令 `q_upper`，同时输出中间控制信号 `g' = [rpy, h]`，也就是torso roll/pitch/yaw和base height  
AMO module `phi(q_upper, rpy, h) -> q_ref_lower`  
它根据上身姿态和torso/height命令，输出下半身关节参考 `q_ref_lower`，这是通过trajectory optimization数据训练出的“下身该怎么配合上身”的映射  
lower policy `π'_lower(v, g', s_proprio) -> q_lower`  
它输入base速度、torso/height中间命令、本体状态，并结合AMO提供的lower-body reference，输出下半身关节命令  

注意 `g'` 和 `q_ref_lower` 不是一个东西：  
`g' = [rpy, h]` 是upper policy / multi-target IK求出来的中间身体命令，只包含torso orientation和base height  
`q_ref_lower = phi(q_upper, rpy, h)` 是AMO module输出的下身参考关节角，是15维lower-body reference  
也就是说 `g'` 先被送进AMO module，AMO module再根据 `q_upper + g'` 生成 `q_ref_lower`  
`q_ref_lower` 不直接控制机器人，而是作为lower RL policy的observation/reference，让lower policy知道“为了配合当前上半身和身体姿态，下身大概应该站成什么样”  

AMO里最关键的RL策略是lower policy  
teacher lower policy：`π_teacher(v, g', s_proprio, s_priv) -> q_lower`  
student lower policy：`π_student(v, g', s_proprio, s_hist) -> q_lower`  
其中 `s_priv` 是仿真特权状态，如真实base速度、真实torso rpy和height、接触信息等；student部署时用25步本体历史 `s_hist` 替代特权信息  
`q_lower` 是15维动作：双腿 `2 x 6` 个目标关节位置 + 腰部 `3` 个目标关节位置  
这些目标位置再由PD执行  
上身 `q_upper` 不是这个lower RL policy输出的，而是由VR teleop + retargeting / multi-target IK 或自主上层IL policy产生  
所以AMO最终控制全身，但“全身”来自两个来源：上身目标 `q_upper` 由上层给，下身+腰 `q_lower` 由RL lower policy给  
这点和ULC不一样：AMO是decoupled/hierarchical，ULC是single policy直接输出legs + torso + arms的目标关节位置  

上层命令来源：  
遥操作实验里来自VR teleoperation系统的头/双手pose和base速度命令  
自主实验里作者用HOMIE/AMO系统采集demonstrations，再训练ACT imitation policy；ACT输入双目图像 + 上身本体状态 + 上一时刻命令，输出head/dual-arm/dual-hand joint angles以及`v, rpy, h`，再交给AMO/lower policy执行  
也就是说自主任务是IL policy提供上层命令，不是AMO controller自己带视觉决策  
是否带感知：AMO底层controller主要吃proprioception + command/reference；自主IL policy才吃双目图像  
局限：它解决的是大工作空间稳定控制/遥操作底层，不是端到端视觉闭环；对上层任务仍需要teleop或IL policy提供命令  
链接：https://amo-humanoid.github.io/  

#### FALCON: Learning Force-Adaptive Humanoid Loco-Manipulation
2025 arXiv  
FALCON = Force-Adaptive humanoid Loco-manipulation Controller / Learning Force-Adaptive Humanoid Loco-Manipulation 作者自命名  
任务是人形机器人在真实接触力下做loco-manipulation：搬载荷、拉车、开门  
它关注的是末端执行器受到未知3D外力时，机器人还能不能保持走路和上身跟踪  

方法是dual-agent RL：  
lower-body agent负责在外力扰动下稳定locomotion  
upper-body agent负责精准跟踪双臂/末端目标，并隐式补偿接触力  
两者在仿真中jointly train  
关键设计是force curriculum：训练时逐渐增加施加在end-effector上的外力大小，同时考虑关节力矩限制  
这样policy不需要真实force sensor，也能从proprioception history里学到隐式抗力/抗负载行为  

仿真器：Isaac Gym  
真实机器人：Unitree G1 和 Booster T1  
时间：arXiv 2025-05  
策略输入/输出：FALCON把全身自由度拆成lower-body DoF和upper-body DoF，训练两个RL agent  
shared proprioception `s_p,t = [q_{t-4:t}, qdot_{t-4:t}, omega_root, ...]`，也就是多步关节状态/root角速度等本体历史  
lower policy `π_l : s_p,t x G_l,t -> A_l,t`，输出lower-body joint PD targets  
upper policy `π_u : s_p,t x G_u,t -> A_u,t`，输出upper-body joint PD targets  
最终动作 `a_t = [a_l,t; a_u,t]` 拼起来发给joint-level PD controller  
训练时upper-body target joint angles `q_upper_target` 从AMASS随机采样；部署时上身目标通过IK计算  
critic训练时还能看到privileged root linear velocity和end-effector forces，但部署actor不能看这些  
是否带感知：不带RGB/depth/语言感知；它依赖结构化命令和本体状态  
上层命令来源：teleop时用VR/joystick + IK给上身目标；autonomy pipeline里用FoundationPose估物体pose，再motion planning/IK生成手臂目标  
FALCON controller本身不负责感知，也不直接决定任务目标  
能做什么：transporting payloads 0-20N、cart-pulling 0-100N、door-opening 0-40N  
局限：主要解决力/负载鲁棒性，不是自主视觉决策；通常仍需外部planner / 目标pose / 任务先验来告诉手往哪里去  
链接：https://arxiv.org/abs/2505.06776  

#### R2S2: Unleashing Humanoid Reaching Potential via Real-world-Ready Skill Space
2025 RA-L  
R2S2 = Real-world-Ready Skill Space 作者自命名  
任务是释放人形机器人大范围3D reaching能力：机器人需要同时掌握base移动、转向、身体高度/姿态调整、末端到达不同空间位置  
它能做各种goal-reaching场景，也能支持大工作空间teleoperation  

方法不是从零训练一个大一统policy，而是先训练一组real-world-ready primitive skills：  
locomotion：跟踪base线速度/角速度  
body-pose-adjustment：调身体高度、弯腰、保持脚接触  
hand-reaching：控制手/末端6D pose  
然后把这些异构primitive skills ensemble到统一latent skill space里，通常可理解成用CVAE/latent skill prior形成可采样技能空间  
高层planner再从这个skill space里采样/组合技能，完成复杂goal-reaching  

仿真器：Isaac Gym  
真实机器人：论文称在不同机器人平台、仿真和真实中验证，公开材料里重点是Unitree G1/H1类人形平台  
时间：arXiv 2025-05；IEEE RA-L accepted 2025  
策略输入/输出：不是单一“图像到关节”的VLA，而是skill-space/controller框架  
primitive skill形式是 `π_prim : G_prim x S_prim -> A_prim`  
locomotion skill：goal是线速度/角速度命令，输出对应关节动作  
body-pose-adjustment skill：goal是body height和pitch angle，输出关节动作  
hand-reaching skill：goal是end-effector pose，输出关节动作  
这些动作本质上仍是机器人关节控制目标/低层动作，经PD执行  
第二阶段用CVAE把这些primitive skill ensemble成统一latent skill space：encoder从state/goal到latent `z_t`，decoder从state/latent到joint action `a_t`  
第三阶段high-level planner不是输出电机关节，而是在latent skill space里采样/选择 `z_t`，再由decoder变成低层关节动作  
是否带感知：控制器本身主要用proprioception + goal，不带视觉；视觉/语言任务需要额外上层模块给目标  
上层命令来源：可以来自teleop、人工目标、planner或任务脚本；论文核心不是学习从图像直接产生这些目标  
能做什么：大范围reach、高低不同位置取物/触达、需要移动+弯腰+伸手的目标到达  
局限：结构化skill prior很适合控制，但不是视觉端到端VLA；WholeBodyVLA指出它仍需要额外信息/输入来设定目标或技能，任务层闭环自主性有限  
链接：https://arxiv.org/abs/2505.10918  

#### ULC: A Unified and Fine-Grained Controller for Humanoid Loco-Manipulation
2025 arXiv  
ULC = Unified Loco-Manipulation Controller 作者自命名  
任务是用一个单一end-to-end policy同时做人形机器人的移动和双臂跟踪  
它想反驳“必须把上身manipulation和下身locomotion拆成两个policy”的思路  

方法核心是单策略统一跟踪多个目标：root velocity、root height、torso rotation、dual-arm joint positions  
为了让单策略能训稳，作者加入 sequence skill acquisition、residual action modeling、command polynomial interpolation、random delay release、load randomization、center-of-gravity tracking  
其中residual action modeling用于细粒度调整，polynomial interpolation让命令切换更平滑，load randomization提高外部负载鲁棒性，CoG tracking给稳定性更直接的梯度  

仿真器：Isaac Lab / Isaac系并行RL训练环境  
真实机器人：Unitree G1 with 3-DoF waist  
时间：arXiv 2025-07  
策略输入/输出：ULC训练的是一个single unified policy `π_theta : S x G -> Δ(A)`  
它是高斯策略，输入state-command observation，输出action distribution  
observation是纯本体：`o_prop(t) = [q_joint(t), qdot_joint(t), omega_base(t), g_proj(t), a_{t-1}, g_t]`  
完整输入还会stack多个历史timestep  
command `g_t = [g_loco, g_torso, g_arms]`  
`g_loco = [v_xy, omega_z, h_pelvis]`，也就是root平面速度、yaw角速度、pelvis/root高度  
`g_torso` 是torso ZXY Euler angles，也就是yaw/roll/pitch目标  
`g_arms` 是左右手臂的目标关节位置，不是EE pose  
action就是所有actuated DoF的target joint positions：  
`a_t = [q_target_legs, q_target_torso, q_target_arms]^T * alpha_scale + q_default`  
其中 `alpha_scale = 0.25`，最后由joint-level PD controller执行  
所以ULC不是只输出腿，也不是只输出残差速度；它的单个RL policy同时输出腿、3-DoF腰/torso、双臂的目标关节位置  
arm部分用了residual action modeling：  
先有policy processed action `q_processed = alpha_scale * π_theta(s,g) + q_default`  
再对upper-body/arm joints做 `q_final[J_upper] = q_processed[J_upper] + q_desired[J_upper]`  
这里 `q_desired` 来自command interpolation / delay机制生成的arm desired joint positions  
所以手臂最终PD目标 = 用户/任务给的手臂目标关节 + policy学出来的残差修正  
是否带感知：不带视觉/语言感知，是command-conditioned controller  
上层命令来源：双臂关节命令、身体高度、torso姿态、root速度由teleop/planner/task script给出；论文训练中这些commands是procedurally sampled并用五次多项式插值/随机release模拟部署命令变化  
ULC负责稳定跟踪这些命令，不负责从相机或语言自主生成任务意图  
能做什么：更统一的dual-arm tracking + walking/squatting/torso控制，在外部负载下保持协调；相比分离式方法有更大workspace和更好tracking  
局限：虽然比分层方法更统一，但仍是跟踪命令的controller，不是自己从视觉和语言生成任务策略；长程任务仍需要上层policy/planner给命令  
链接：https://arxiv.org/abs/2507.06905  

#### 这组工作的共同边界
这几篇都在把humanoid whole-body controller往loco-manipulation推进  
但很多工作本质还是“稳定执行别人给的命令”：速度、身体高度、手臂pose、关节轨迹、skill latent、外力扰动等  
它们真正强的是低层稳定性、遥操作可控性、大工作空间、负载鲁棒性  
不强的是：从相机/语言自主决定何时走、何时停、怎么朝向、怎么抓、失败后怎么重新规划  

所以WholeBodyVLA批评的点可以理解为：  
velocity-tracking接口对巡航足够 但对loco-manipulation不够精确  
因为操作任务需要episode-level controllability，比如停在物体前多远、刹车是否准、身体朝向是否适合抓取、拿起重物时步态是否连续  
如果低层只学逐步速度误差，上层VLA会很难稳定地调用它完成长程任务  

### High-level Planners / VLA for Humanoid Loco-Manipulation
这组工作和上面的controller论文关系很直接：  
低层RL controller通常不吃RGB和语言，所以不能自己做自主任务  
于是另一条路线是：用VLM/LLM/高层planner把图像和语言转成skill command、latent command或motion command，再交给低层whole-body controller执行  

这类方法大概分两派：  
1）modular planner路线：VLM/LLM负责分解任务和选择skill，低层skill library执行  
优点是模块清楚、容易调试；缺点是skill边界脆弱，比如导航结束的位置不适合操作，或者切换skill时身体姿态不稳定  
2）VLA / latent action路线：把视觉语言直接映射到动作token、latent verb或未来motion，试图减少手写skill边界  
优点是更统一；缺点是数据量、实时性、低层稳定性和真实泛化都很难  

#### Humanoid-VLA: Towards Universal Humanoid Control with Visual Integration
2025 arXiv  
Humanoid-VLA是早期把vision-language-action思想扩到humanoid上的工作  
它主要想解决的问题是：人形机器人控制框架大多是reactive controller，缺少语言理解、第一人称视觉和自主交互能力  

方法pipeline：  
先做language-motion pre-alignment，用非第一人称human motion dataset和文本描述对齐，让模型学动作语义  
再做video-conditioned fine-tuning，把egocentric visual context加进来，让motion generation能根据当前场景调整  
还用self-supervised data augmentation从motion data里自动生成pseudo QA annotations，把原始motion sequence转成语言监督  

输入输出：  
输入是language instruction + egocentric video / visual context  
输出不是低层电机力矩，而是面向whole-body control architecture的motion/control representation  
论文摘要层面说它built upon whole-body control architectures，也就是仍然需要底层WBC来执行生成的motion  
所以它不是“RGB+语言直接到全部电机target”的完整低层控制器  

是否带感知：带第一人称视觉和语言  
是否遥操作/动捕：训练依赖human motion datasets和文本/伪标注；不是纯真实机器人遥操作数据  
任务能力：更偏locomotion、object interaction、environment exploration，强调上下文感知和motion generation  
局限：相比GR00T/WholeBodyVLA这类更强系统，它更偏humanoid locomotion / motion侧，manipulation和真实loco-manipulation闭环能力不完整  
链接：https://arxiv.org/abs/2502.14795  

#### GR00T N1: An Open Foundation Model for Generalist Humanoid Robots
2025 NVIDIA arXiv  
GR00T N1是NVIDIA提出的generalist humanoid robot foundation model  
它不是专门为loco-manipulation / whole-body navigation设计，而是更偏generalist humanoid manipulation foundation model  

方法是dual-system VLA：  
System 2：vision-language module，理解环境图像和语言指令，产生上下文embedding  
System 1：diffusion transformer action generator，根据System 2 embedding、本体状态和噪声action，通过flow matching / diffusion方式生成平滑动作片段  
训练数据是heterogeneous mixture：真实机器人轨迹、人类视频、合成数据  

输入输出：  
输入是视觉观测 + 语言指令 + robot proprioception  
输出是action chunks / motor actions  
对单臂可以是EE pose和gripper state；对bimanual / humanoid可以是双臂、灵巧手、torso、neck等关节角或速度  
在论文真实部署里重点是Fourier GR-1上的language-conditioned bimanual manipulation  

是否带感知：带视觉和语言  
是否遥操作/动捕：训练混合真实机器人demonstrations、人类视频和synthetic data，不是纯RL仿真  
任务能力：bimanual manipulation、pick-place、handover、container insertion/closing等  
局限：主要展示桌面/局部双臂操作，不强调移动、转身、下蹲、推车这类全身loco-manipulation；如果用在移动操作，还需要额外whole-body controller / locomotion接口  
链接：https://arxiv.org/abs/2503.14734  

#### Being-0: A Humanoid Robotic Agent with Vision-Language Models and Modular Skills
2025 arXiv  
Being-0是典型modular planner路线：Foundation Model负责高层认知，Connector负责把语言计划翻译成可执行skill command，底层skill library负责稳定执行  
它的目标是让full-sized humanoid在真实室内环境里完成long-horizon household tasks  

三层结构：  
Foundation Model：部署在cloud，负责instruction understanding、high-level reasoning、task planning  
Connector：部署在onboard，是轻量VLM，负责根据第一人称视觉和计划触发具体skill command，并协调navigation和manipulation  
Modular Skill Library：部署在onboard，包含locomotion和dexterous manipulation技能  

输入输出：  
系统输入是语言指令 + active vision / egocentric visual observations  
FM输出高层语言计划  
Connector输出actionable skill commands，比如导航到哪里、调整姿态、调用哪个抓取/放置skill  
底层skill再输出具体机器人控制命令  
所以Being-0不是一个single policy直接输出电机，而是VLM planner + skill library  

是否带感知：带active vision和语言；作者强调active camera重要，固定相机会导致导航和操作失败  
是否遥操作/动捕：论文重点不是MoCap全身控制，而是VLM planner连接已有skill library；具体skill本身可能来自各自训练/工程模块  
任务能力：Fetch Bottle、Deliver Basket、Prepare Coffee、Make Coffee、Deliver Coffee，以及Grasp Bottle、Open Beer、Play Chess等技能演示  
局限：cloud FM引入延迟；模块化skill边界脆弱，导航结束时如果姿态/距离不适合操作，就需要额外pose adjustment；整体能力很依赖已有skill library质量  
链接：https://arxiv.org/abs/2503.12533  

#### LeVERB: Humanoid Whole-Body Control with Latent Vision-Language Instruction
2025 arXiv  
LeVERB = Latent Vision-Language-Encoded Robot Behavior 作者自命名  
它是这组里和whole-body control结合最紧的一篇：不是让VLM直接选择离散skill，而是学习一个latent verb作为高层视觉语言和低层WBC之间的接口  

方法pipeline：  
LeVERB-Bench：用retargeted human MoCap在photorealistic IsaacSim环境中生成视觉语言数据和闭环benchmark  
LeVERB-VL：10Hz高层vision-language policy，输入视觉和语言，输出latent verb `z_t`  
latent vocabulary：用residual CVAE学习连续latent action vocabulary，使语言/视觉能对齐到可执行whole-body motion objective  
LeVERB-A：50Hz低层RL WBC，输入latent verb和本体反馈，输出joint-level / dynamics-level whole-body commands  

输入输出：  
高层输入：egocentric visual observation + language instruction  
高层输出：latent verb `z_t`，不是手写skill id，也不是EE pose/root velocity  
低层输入：`z_t` + proprioception  
低层输出：whole-body joint-level commands / dynamics-level commands，经低层控制执行  
这点是它相对Being-0/R2S2这类skill planner的主要区别：高层动作接口是学习出来的latent，而不是人工定义的skill边界  

是否带感知：带视觉和语言  
是否遥操作/动捕：训练数据来自synthetic rendering + retargeted human MoCap；真实Unitree G1 zero-shot部署，没有真实finetune  
任务能力：LeVERB-Bench 150+任务、10类whole-body WBC任务；简单视觉导航约80%成功，overall 58.5%；真实G1展示根据语言和视觉做semantically-guided whole-body behavior  
局限：仍依赖synthetic/MoCap数据；overall成功率说明复杂任务还远没解决；future work包括更长时序规划、更快视觉反馈、把视觉直接接进低层控制、多机器人泛化  
链接：https://arxiv.org/abs/2506.13751  

#### R2S2: Real-world-Ready Skill Space
2025 RA-L  
R2S2前面已经按controller/skill-space整理过  
放在high-level planner这条线里看，它的定位是：把locomotion、body-pose-adjustment、hand-reaching等primitive skill编码进统一latent skill space，高层在latent里采样或组合  

输入输出关系：  
高层planner输出latent skill `z_t` 或选择/组合primitive skill  
decoder / controller再把latent变成joint action  
它不直接吃RGB/语言，所以如果要做自主任务，还需要额外VLM/planner给目标或latent  

和LeVERB的区别：  
R2S2的latent skill更偏控制/skill library抽象  
LeVERB进一步把视觉和语言接到latent verb上，使latent直接成为VLA和WBC之间的接口  

#### HEAD: Hand-Eye Autonomous Delivery: Learning Humanoid Navigation, Locomotion and Reaching
2025 CoRL  
HEAD = Hand-Eye Autonomous Delivery 作者自命名  
它的核心不是语言，而是从人类视觉和运动数据中学习“眼睛和手应该送到哪里”  
目标是让humanoid在复杂室内环境中导航到用户指定目标，并把手/眼送到目标附近完成reaching  

系统两层：  
High-level policy：输入egocentric RGB图像和2D goal point，输出未来eyes/head和hands的目标pose  
Navigation module：用DINO视觉特征和Transformer decoder预测一串6D camera poses，也就是未来眼睛/头部轨迹  
Reaching module：当目标进入downward RGB-D camera视野后，用3D目标位置和IK生成手/头目标pose  
Low-level WBC：输入三个稀疏6D目标，即eyes、left hand、right hand的目标位置和朝向；输出全身关节控制，使机器人走、转、弯腰、伸手并保持平衡  

输入输出：  
高层输入：RGB / RGB-D + 用户在初始图像上点选的目标  
高层输出：eyes/head + left/right hand的6D target poses  
低层输入：这三个6D target poses + 本体状态  
低层输出：whole-body joint actions / joint targets，经低层控制执行  
所以HEAD的接口非常清楚：高层不输出速度，也不直接输出全身关节，而是输出“眼睛和双手三点的目标pose”  

是否带感知：带视觉，但不强调自然语言；通常需要用户选目标点  
是否遥操作/动捕：低层从大规模human MoCap学习三点tracking；高层从Aria glasses人类第一人称数据学习，还用ADT数据和少量机器人数据  
任务能力：机器人在真实复杂室内环境中自主导航并reaching到指定目标  
局限：任务主要是navigation + reaching，不是完整双臂物体操作；仍有模块边界，高层给三点目标，低层执行；需要目标选择/goal point，不是任意语言任务规划  
链接：https://stanford-tml.github.io/HEAD/  

#### Boston Dynamics LBM / Atlas Demonstration
2025 Boston Dynamics blog/demo  
Boston Dynamics展示的是Atlas上的Large Behavior Model / language-conditioned policy路线  
它不是传统论文式完整开源方法，但很有参考价值，因为它展示了工业级humanoid把teleoperation、MPC、VLA-like policy和全身loco-manipulation结合起来的方向  

方法信息：  
policy输入是images、proprioception和language prompts  
policy输出控制full Atlas robot的actions，频率约30Hz  
模型使用diffusion transformer和flow matching loss  
数据来自高质量VR teleoperation系统，teleop系统结合Atlas MPC和VR接口，可以覆盖手指级灵巧、全身reaching和locomotion  

输入输出：  
输入：多视角/头部相机图像 + proprioception + language prompt  
输出：和teleoperation系统相同的控制接口，控制全Atlas执行全身动作  
不是简单skill id，也不是只控制上半身  

任务能力：Spot Workshop长程任务，包括走到货架/推车、调整站姿、蹲下、抓取Spot零件、regrasp、放置、拉出bin、处理物体掉落等  
局限：强依赖昂贵高质量teleoperation/MoCap-like数据采集和Atlas专有MPC/硬件栈；展示环境和任务分布仍相对受限；不是开源可复现实验  
链接：https://bostondynamics.com/blog/large-behavior-models-atlas-find-new-footing/  

#### 这组工作的逻辑关系和我的判断
从低层controller到高层planner/VLA，大概是这样演进的：  
HOMIE / AMO / FALCON / ULC：先把人形机器人变成“能稳定执行命令”的机器  
R2S2：把可执行命令包装成latent skill space，让高层更容易组合  
Being-0：用VLM/LLM planner调skill library，解决长程语义任务，但skill边界和cloud latency是问题  
HEAD：不用语言，改用视觉目标点和三点hand-eye接口，把导航和reaching连起来，接口很干净  
Humanoid-VLA / GR00T：走VLA基础模型路线，把视觉语言和动作生成合在一起，但一个偏motion/locomotion，一个偏manipulation  
LeVERB：试图用latent verb把VLA和低层WBC真正接起来，是和WholeBodyVLA最接近的前作之一  
Boston Dynamics LBM：工业展示说明这条路线可行，但代价是专有硬件、MPC和大规模高质量teleop数据  

共同问题：  
模块化planner的短板是skill boundary，机器人走到某个位置后身体姿态不一定适合操作  
端到端VLA的短板是数据和低层稳定性，动作空间太高维，真实收集成本极高  
所以WholeBodyVLA这类工作的核心卖点就是：把视觉语言和whole-body low-level control之间的接口做成统一latent/action representation，尽量减少“先导航停下 再操作”的割裂  

### Visual Whole-Body Control for Legged Loco-Manipulation
CoRL 2024 UC San Diego  
VBC = Visual Whole-Body Control 作者自命名  
任务是 Unitree B1 + Unitree Z1 arm 做视觉自主pick-up：机器人从自我视角看到目标物体后，自主走近、调整身体高度/姿态、伸臂抓取不同高度的物体  
这篇是DQ-Net继承最多的前作：低层whole-body goal-reaching policy + 高层teacher/student视觉策略 + approach/progress/completion分阶段reward  

#### 解决什么问题
普通四足机械臂如果只用默认base controller，机身高度基本固定，机械臂可达空间有限  
比如地面上的物体、低箱子上的物体、桌面上的物体，需要机器人弯腰、抬身、靠近、稳定身体才能抓  
VBC的目标不是动态物体，而是静态物体pick-up，但强调whole-body：腿不是只负责移动，而是和机械臂一起扩大操作空间  

和之前很多工作相比，它不依赖mocap或人工遥操作目标位姿  
真实部署时只需要用户一开始用TrackingSAM点选/分割目标，之后机器人用两路RGB-D的mask和masked depth自主抓取  

#### 总体pipeline
三阶段训练：  
1）训练通用低层policy π_low：跟踪base速度命令和EE目标pose，实现全身goal-reaching  
2）冻结低层policy，训练privileged teacher high-level policy：输入物体pose/shape等特权状态，输出高层命令完成pick-up  
3）用DAgger把teacher蒸馏成visuomotor student：student只看视觉mask/depth和本体状态，用于真实部署  

部署时：  
student high-level policy约6-8Hz输出高层命令  
low-level policy 50Hz控制四足腿部  
机械臂IK controller约800Hz根据目标EE pose控制Z1 arm  
高层更新之间，低层和机械臂继续使用上一次高层命令  

#### Low-level policy
低层policy目标是“全身跟踪器”，不是具体任务策略  
低层命令 b_t = [p_cmd, o_cmd, v_cmd_lin, omega_cmd_yaw]  
p_cmd/o_cmd 是EE位置和姿态目标，v_cmd_lin是base前进速度，omega_cmd_yaw是base yaw速度  

低层观测90维：  
base state：roll/pitch/yaw velocity  
arm state：机械臂关节位置/速度  
leg state：腿关节位置/速度和足端接触  
上一时刻action  
环境外参latent z_t：编码摩擦、质量、质心等物理参数  
gait timing reference  
当前低层command  

低层输出：12个腿关节target angles，经过PD控制  
机械臂不由低层RL直接输出关节，而是用IK把EE目标pose转成Z1 arm目标关节  
作者认为直接让RL输出arm joint不够精确，所以用“EE目标 + IK”提高末端控制精度  

低层训练用PPO，并采用ROA（Regularized Online Adaptation）/RMA式外参适应  
训练时用privileged physics参数编码latent，部署时用历史本体观测估计latent  
低层reward包括command following、energy penalty、alive reward、phase reward  
具体包括base速度/yaw tracking、角速度惩罚、关节力矩/action rate、碰撞、feet air time、默认关节位置、z向速度、base height、swing/stance phase tracking等  

#### High-level teacher
teacher是状态策略，用RL训练，底下接冻结的low-level policy  
teacher输入1094维：  
z_shape：PointNet++从物体点云编码出的1024维shape feature  
s_obj：物体在arm base局部坐标系下的6D pose  
s_proprio：机器人本体状态，包括19维关节位置、18维关节速度、EE pose等  
v_base：base velocity  
上一时刻高层action  

teacher输出9维高层action：  
p_cmd_t：6维gripper pose increment  
v_cmd_t：2维base linear velocity + yaw velocity  
p_gripper_t：1维gripper open/close  

注意这里的高层不是直接输出完整电机动作，而是给低层一个“身体怎么动 + 手往哪伸 + 夹爪开合”的命令  
这个设计使低层policy可以复用到不同高层任务里  

#### Visuomotor student
student用于真实部署，输入是：  
两路相机的object segmentation masks + masked depth images  
机器人proprioception  
上一时刻action  

两路相机：head/front camera + wrist camera，都是RealSense D435  
作者不用RGB作为策略输入，主要用mask和masked depth，目的是减少sim-to-real gap  
真实中需要用户在每次reset开始时用TrackingSAM点选目标，之后TrackingSAM约10Hz持续给mask  
只要目标在head或wrist任一相机可见，mask可以传播到两个视角  

student训练用DAgger：  
先由teacher采样初始数据warm-up  
然后用student rollout，遇到状态就向teacher请求correct action  
loss是student action和teacher action之间的MSE  
作者明确说这样比直接用视觉RL训练student更高效稳定  

网络结构：  
学生策略把4步历史图像stack起来，经过两层CNN编码成64维latent  
再和本体状态/上一时刻action拼接，过两层MLP输出9维高层action  

#### High-level reward
高层task reward是三阶段设计，这部分很重要，因为DQ-Net也沿用这个结构  
每个时刻只处于一个stage，只激活一个stage reward，避免阶段目标互相干扰  

Stage 1 approaching：  
鼓励gripper和quadruped接近物体  
rapproach = min(d_closest - d, 0)  
d_closest是当前已经达到过的最小gripper-object距离，d是当前距离  
如果当前距离比历史最近更近，reward接近0或变好；如果变远则负  

Stage 2 progress / lifting：  
pickup任务里鼓励物体高度提升  
r_progress_pickup = min(d - d_highest, 0)  
d_highest是当前已经达到过的最高物体高度，d是当前高度  
所以它奖励继续把物体抬得更高  

Stage 3 completion：  
r_completion = 1(Task is completed)  
pickup任务里完成条件是物体高度超过阈值  
真实实验里成功定义为物体被抬起超过放置平面0.1m  

权重：rapproach 0.5，rprogress 1.0，rcompletion 3.5  

辅助reward：  
racc：限制arm joint velocity变化  
rcmd：鼓励靠近物体时降低base速度  
raction：平滑高层action  
ree_orn：EE朝向物体  
rbase_orn：base朝向物体  
rbase_approach：base接近物体，目标大约保持0.6m距离  

关键训练技巧：  
action delay：模拟真实推理/执行延迟  
command clip curriculum：先允许较大base速度加速学习，后期逐渐clip让真实部署更安全  
reward curriculum：等会抓后再加入速度命令惩罚  
randomly changing object position：10%概率随机改变物体pose，让策略学会失败后retry  
forcing stop when closing gripper：夹爪闭合时强制base velocity为0，形成stop-then-pick行为  

#### 失败和retry机制
VBC允许retry，这和DQ-Net的一次抓取设定不同  
仿真里episode成功：物体被pick up  
失败：物体掉下桌面/放置面，或150个high-level steps内没成功  
真实实验里一个object-height配置最多允许5次连续attempts，超过5次或物体掉落算failure  

作者观察到emergent retrying behavior：第一次没抓住时，只要mask没有丢，机器人会重新调整身体和手臂继续抓  
retry成功的原因包括：早期尝试改变了物体pose，以及机器人身体移动改变了抓取视角/可达性  
这点对静态物体pick-up很有帮助，但也说明它和动态抓取benchmark的目标不同：VBC允许反复尝试，DQ-Net用小平台和OSSR强调one-shot  

#### Sim-to-real
仿真器：Isaac Gym  
硬件：Unitree B1 + Unitree Z1 arm + two RealSense D435 + B1 onboard computer + Jetson Orin  
训练对象：仿真用33/34个YCB objects，按ball/long box/square box/bottle/cup/bowl/drill分类  
真实对象：14个物体，包含规则物体、日用品、irregular objects，大部分真实物体仿真中未见过  
真实任务高度：地面0.0m、箱子约0.3m、桌子约0.5m  
真实物体初始放在机器人前方约0.5-1.5m，保证至少一个相机能看到  
所有策略仿真训练，真实没有人工采集数据或finetune  

视觉sim-to-real策略：  
不用RGB，只用mask + segmented/masked depth  
相机pose随机化  
depth clipping到最小0.2m，hole filling，normalize  
RandomErasing、GaussianBlur、GaussianNoise、RandomRotation等增强  
student训练时随机camera latency，模拟真实图像延迟  

#### 实验结果
仿真：VBC student显著优于floating base和non-hierarchical baseline  
non-hierarchical end-to-end policy训练失败，说明直接把视觉和低层全身控制端到端学出来很难  
floating base没有真正arm-leg coordination，低处物体/地面物体表现很差  
VBC在不同物体类别和高度上整体最好，但student明显弱于teacher  

teacher-student gap在Table 1里很明显：  
VBC privileged teacher在7类物体上的平均成功率约84.4%  
VBC visuomotor student平均约58.7%  
绝对下降约25.7个百分点  
按类别看：Ball 89%->67%，Long Box 93%->42%，Square Box约81.9%->74%，Bottle 76%->55%，Cup 89%->74%，Bowl 78%->52%，Drill 84%->47%  
下降最严重的是Long Box和Drill 说明只靠mask/depth student很难恢复teacher通过3D shape feature和object pose获得的精细形状/姿态信息  
作者也说student第一次接触但没抓住后 物体姿态会变得更难 只有视觉输入且150 high-level steps限制下更难补救  

一个有意思的现象：  
teacher用3D shape feature对复杂/不规则物体帮助很大，比如long box、cup、bowl、drill  
但对简单规则物体，有时不加shape feature也能抓得不错，因为抓中心就够了  

真实：VBC在地面、箱子、桌面三种高度都优于teleoperated low-level baseline  
baseline在0.0m和0.3m几乎失败，因为默认controller机身高度固定，手臂够不到或姿态不合适  
VBC的优势来自全身弯腰/抬身/靠近，让机械臂进入可达空间  

#### limitation / future work
最常见失败原因是TrackingSAM mask tracking loss  
如果机器人转身/倾斜、目标被遮挡、目标颜色和环境相似，mask会丢或混淆  
当前系统每次任务开始还需要人工点选目标，不是完全自主搜索  

深度估计不准，尤其是反光物体  
Z1自带beak-like gripper不够好，容易把物体推走，不如平行夹爪适合精确抓取  
pipeline层级多，任一模块误差都会累积：mask、depth、student、低层tracking、IK、夹爪都会造成compounding error  
输入不含RGB颜色，所以不能做依赖颜色语义的任务，比如区分Fanta/Sprite或按绿色按钮  
未来工作包括：自动目标检测/搜索探索模块、更好的gripper和RGB-D相机、更多物体尺寸随机化、简化整体pipeline  

#### 和DQ-Net关系
DQ-Net基本继承了VBC的分层控制思想和三阶段高层reward  
VBC解决静态物体、多高度、多真实物体pick-up，并且允许retry  
DQ-Net把任务推进到动态物体抓取，加入GFM抓取先验、object velocity、动态平台benchmark，但暂时没有真实部署  
所以可以把VBC看成真实静态loco-manipulation基线，把DQ-Net看成仿真动态抓取扩展  

### Whole-Body Coordination for Dynamic Object Grasping with Legged Manipulators AAAI2025
这篇核心贡献有两个：一个是 DQ-Bench（Dynamic Quadruped Grasping Benchmark 可以理解为作者自建的动态四足抓取benchmark） 另一个是 DQ-Net（Dynamic Quadruped Grasping Network 作者自命名的teacher-student策略框架）  
任务是 Unitree B1 + Z1 arm 在IsaacGym里抓运动物体 物体不是静止放在桌上 而是在一个很小的floating platform上运动 通过平台摩擦/法向力带动物体运动 这样比直接给物体施加随机力更物理合理  
这篇没有做真实机器人部署 主要是仿真benchmark和仿真实验  

#### DQ-Bench任务设置
对象来自YCB object dataset（Yale-CMU-Berkeley Object and Model Set 常用机器人抓取物体集） 一共43类 30类seen训练/测试都用 13类unseen只测试 用来测泛化  
难度分四级：  
Level 1 低速0-15cm/s 固定轨迹 line/arc  
Level 2 高速15-30cm/s 固定轨迹  
Level 3 高速0-30cm/s 随机2D轨迹  
Level 4 还加入z轴自由运动 变成随机3D轨迹  
每个episode物体初始高度随机0.2-0.7m 地形有0-0.1m随机起伏  
指标：GSR（Grasp Success Rate 抓取成功率） OSSR（One-Shot Success Rate 一次抓取成功率） TSC（Time Steps to Completion 完成抓取步数）  
他们还把平台做得只比物体大一点 避免机器人训练时利用大平台作弊 比如先把物体拨到平台上再慢慢调整  

#### 总体控制框架
不是一个policy直接输出所有电机 而是分层：HLP（High-Level Policy 高层策略） + LLP（Low-Level Policy 低层whole-body tracking policy）  
HLP输出高层动作：EE（End-Effector 末端执行器）position increment ∆p in R3 EE orientation increment ∆r in R3 base linear velocity vlin 和 yaw rate wyaw  
这些会积分成目标EE pose 再和base速度命令组成低层命令 u = [p_ee, r_ee, vlin, wyaw] 维度8  
LLP接收低层命令 + 机器人本体状态 + terrain/environment embedding 输出12个腿关节目标角度 通过PD控制腿  
机械臂不是由LLP网络直接输出关节 而是对目标EE pose做IK 得到Z1 arm目标关节  
所以最后执行动作是 leg target joint angles + arm IK joint angles  

低层LLP是先用PPO训练好再冻结的 目标是稳定跟踪base速度和EE pose  
低层观测包括 base states arm joint states leg joint states previous action 环境参数zt gait phase 当前command  
环境参数zt对应训练中随机化的mass/friction/inertia等 类似RMA里的外部状态编码 提高domain robustness  
低层reward主要是速度跟踪 yaw跟踪 能耗/action smoothness/collision/base height/gait phase等  

#### Teacher policy怎么做
teacher用privileged信息训练高层策略 输入包括：object point cloud feature fp object 6D pose xt object velocity vt fused grasp gt robot proprioception bt  
输出就是上面说的高层动作 a = [∆p, ∆r, vlin, wyaw]  
teacher的关键模块是GFM（Grasp Fusion Module 抓取融合模块 作者自命名） 用来避免每一步都跑重型抓取网络  
这里teacher确实有很多privileged信息 作者明确写了ground-truth object pose / velocity 以及object point cloud feature 但没有细讲teacher点云到底是完整mesh采样 还是仿真相机渲染出的局部点云  
从复现角度看 可以理解成仿真里直接拿目标物体的privileged point cloud 然后用PointNet编码 student部署时并不能拿这个点云  

GFM分两阶段：  
第一阶段是grasp memory construction 离线/预处理 对物体在固定相机视角下用Contact-GraspNet（一个现成的6D抓取位姿预测网络）生成N个6D grasp candidates 然后取top-K 论文实验K=30  
Contact-GraspNet具体预测的是夹爪的6D抓取位姿和质量分数 输入通常是深度图/点云 输出不是一个唯一答案 而是一组 grasp candidates  
每个candidate可以理解成 gripper pose + score：位置xyz + 姿态roll/pitch/yaw或rotation matrix + 这个抓法的质量分数  
为什么会有很多候选：抓取本来就是多解 一个杯子/盒子/瓶子都可能从上方抓 从侧面抓 斜着抓 夹不同部位 所以Contact-GraspNet会在可见点云表面生成多个可能接触/夹取姿态  
这些候选抓取位姿会转成物体局部坐标系下的relative transform 存入memory bank  
这一步是离线的原因是：刚体物体虽然在世界里不断运动 但“相对于物体自身”的抓取方式是不变的 比如“夹杯身侧面”在杯子坐标系里一直是同一个抓法  
所以动态过程中不需要每一帧重新跑Contact-GraspNet 只需要用当前object pose把object-frame grasp变换到world frame 大概就是 G_world(t) = T_object_world(t) * G_object_local  
第二阶段是dynamic fusion 在线阶段 PointNet把当前物体点云编码成128维feature 再和当前object pose拼起来过MLP得到query  
memory里的相对抓取位姿根据当前object pose变换到world frame 然后每个candidate grasp过MLP得到key/value  
attention用query对K个候选打分 最后加权得到一个fused grasp representation/pose gt  
gt再和object pose velocity point cloud feature proprioception一起输入MLP 输出高层动作  
我的理解：GFM相当于把静态抓取网络的多候选结果缓存成“物体坐标系里的抓法库” 然后在线根据当前物体位姿/速度/点云特征动态选择或融合一个更合适的抓取先验  
论文说grasp memory construction是在fixed camera pose下做的 附录又说Contact-GraspNet在IsaacGym里生成multi-view/multi-pose grasp predictions 但没有明确这个fixed camera是不是student的base/wrist相机  
我倾向于认为这只是离线生成grasp memory用的仿真相机视角 不等同于student部署时的双相机 因为生成后候选都被转到object local frame了 后续和相机视角关系不大  
复现时比较合理的做法是：用若干固定虚拟相机或物体mesh为每个物体离线生成候选抓取 转成object frame存起来；teacher在线拿privileged object pose/velocity/point cloud；student只拿base/wrist的mask+depth  

teacher训练：先冻结LLP 再用RL（Reinforcement Learning 强化学习）训练HLP 训练在最难的Level 4上进行 6000 parallel env rollout length 24 总80k timesteps  
reward沿用/修改VBC（Visual Whole-Body Control 他们对比/继承的静态视觉全身控制方法）静态抓取的分阶段奖励：approach lifting completion 三个阶段一次只激活一个  
辅助reward包括动作平滑 加速度/关节速度 EE朝向对齐 base朝向对齐 base接近物体 base height constraint yaw angle penalty等  
这里加了yaw约束：偏航超过60度给二次/非线性惩罚 超过70度直接early terminate 提高采样效率  

#### Reward和抓取失败怎么处理
这篇的reward要分清楚两层：  
LLP低层reward训练的是稳定跟踪和运动能力 不是直接训练抓取决策  
HLP高层reward才对应“怎么接近物体、怎么抓、怎么完成任务”  

低层LLP reward：  
主要是base线速度/yaw速度tracking 加上角速度惩罚 关节力矩惩罚 action rate 碰撞惩罚 feet air time 默认关节位置 base height swing/stance phase tracking等  
这部分作用是让狗稳定地跟踪高层给的base velocity/yaw和EE pose command  

高层HLP reward：  
作者说沿用VBC的静态抓取reward结构 并做动态任务适配  
任务奖励是staged design 三阶段一次只激活一个：  
1）rapproach：approach阶段 主要鼓励base/EE接近目标物体和合适抓取姿态  
2）rlift：lifting/progress阶段 物体被抓住后鼓励抬起/推进到成功状态  
3）rcompletion：completion阶段 完成抓取给较大奖励  
权重分别是 rapproach 0.5 rlift 0.8 rcompletion 3.5  
主文和附录没有把这三个任务奖励的完整公式展开 只明确了它们来自VBC的分阶段抓取奖励 因此复现时需要看VBC原文或代码  

辅助奖励包括：  
racc：关节速度/加速度变化惩罚 鼓励平滑  
rcmd：限制过大的base前进速度命令  
raction：action变化惩罚  
ree_orn：EE朝向和目标方向对齐  
rbase_orn：base朝向目标  
rbase_approach：base接近物体 公式里希望base和object保持约0.6m距离  
rbase_h：base height constraint 保持适合抓取的机身高度  
ryaw：yaw偏差超过60deg开始惩罚 超过70deg early terminate  

抓取具体怎么发生：  
HLP输出EE位置/姿态增量和base速度/yaw命令  
LLP负责腿部稳定跟踪 机械臂部分不是RL直接输出关节 而是把HLP累计/更新后的target EE pose通过IK求Z1 arm目标关节  
因此抓取过程可以理解成：teacher用GFM给的fused grasp pose作为抓取先验 HLP不断调整base和EE目标pose 让夹爪对准运动物体 然后靠任务阶段reward促成接近-接触/夹取-抬起/完成  

如果刚开始没抓到怎么办：  
DQ-Bench刻意把floating platform做得只比物体稍大 防止机器人把物体轻轻推在大平台上慢慢调整这种cheating  
如果抓取扰动太大或没夹稳 物体会从小平台上掉下去 论文说这会reset scene  
所以训练/评估鼓励one-shot dynamic grasp 而不是多次推、拨、慢慢重新调整  
这也对应OSSR（One-Shot Success Rate）指标：统计一次尝试直接成功的比例  

但论文没有细写“夹爪什么时候闭合/抓取失败后episode是否立即终止”的完整实现细节  
从文字能确定的是：物体掉下平台会reset scene；yaw过大有early termination；成功由GSR/OSSR/TSC统计  
所以复现时需要自己定义清楚contact/lift/success判据 比如夹爪闭合后物体是否被稳定持有并离开平台/达到高度阈值 以及object fall/off-platform是否作为failure termination  

#### Student policy怎么做
student不使用object pose velocity point cloud 只用机载可获得的视觉和本体状态 去模仿teacher动作  
两个视角：base-mounted camera + wrist/end-effector mounted camera  
每个视角输入连续3帧 每帧包括target mask和depth map  
mask来自预训练Track-SAM（基于SAM的目标跟踪/分割工具） 他们认为mask比RGB更少sim-to-real gap depth提供几何信息  
论文附录给的输入tensor是 [B, 12, 54, 96] 12 = 2 modalities(mask/depth) * 2 viewpoints(base/wrist) * 3 frames  
网络是dual-stream transformer：两个视角共享CNN encoder 每帧编码成64维feature  
base stream和wrist stream分开进两个Transformer 每个Transformer 2层encoder 2个attention heads  
proprioception先过MLP成robot state token 拼到visual token前面 同时加temporal positional encoding  
两个stream的输出再投影 concat 过三层FC action head 输出高层动作  
student训练用DAgger（Dataset Aggregation 模仿学习数据聚合方法）/knowledge distillation 冻结teacher 后让student最小化和teacher action的MSE（Mean Squared Error）  
也就是说student不是直接RL从视觉训练 而是行为克隆/蒸馏teacher 这会带来明显teacher-student gap  

#### 观测和输出总结
Teacher HLP输入：物体点云特征fp 当前物体6D pose xt 物体velocity vt GFM输出的fused grasp gt 机器人proprioception bt  
Teacher HLP输出：EE位置增量∆p EE姿态增量∆r base线速度vlin yaw角速度wyaw  
Student HLP输入：base camera连续3帧mask+depth wrist camera连续3帧mask+depth 机器人proprioception bt  
Student HLP输出：和teacher一样的高层动作 用来喂给冻结的LLP和arm IK  
LLP输入：低层command [EE目标pose + base velocity/yaw] + base/arm/leg状态 + 上一时刻动作 + 环境参数zt + gait phase  
LLP输出：12个腿关节target angles 机械臂由IK根据EE目标pose求关节  

#### 实验结论
DQ-Net student在Level 4 GSR 41.0 OSSR 38.5 显著高于VBC-D（作者把VBC改成动态抓取版本的baseline）的16.0/15.2 但teacher在Level 4能到74.3 说明student和teacher之间gap很大  
ablation里去掉GFM会明显掉 去掉velocity在teacher上也掉 说明动态抓取里抓取先验和运动信息都重要  
但是在unseen object上出现过 w/o velocity 有时比full更好 作者解释为显式velocity可能让策略过拟合训练时的运动模式 不一定对未知形状/运动泛化最好  
Transformer student比CNN student更好 参数还更少 5.37M vs 8.43M 说明双视角时序建模有效  

#### limitation / future work
最大限制：没有真实机器人部署 未来工作第一条就是deploy DQ-Net on real quadruped hardware  
student完全依赖teacher蒸馏 在高速/不规则动态场景中 student只有mask/depth和本体信息 很难复现teacher基于精确pose/velocity做出的决策 所以teacher-student gap明显  
作者计划用RL + KD（Knowledge Distillation 知识蒸馏）混合训练 让student在蒸馏后还能通过任务reward试错finetune 缩小gap reward包括抓取成功 动作平滑 能耗等  
GFM现在依赖静态预定义grasp candidate memory bank 虽然attention能动态选择 但候选集合本身固定 对novel objects 或 extreme motion适应性有限  
future work是online learning / self-supervised更新grasp candidate set 在交互过程中持续提取新的grasp pattern 扩展候选池  
另外他们也提到未来考虑deformable/articulated objects multi-object和collaborative manipulation  
复现角度看 这篇最重的依赖是：IsaacGym动态平台环境 YCB物体集 Contact-GraspNet候选生成 Track-SAM mask生成 低层VBC式whole-body tracking policy 以及teacher-student数据采集/DAgger流程

### QuadWBG: Generalizable Quadrupedal Whole-Body Grasping ICRA 2025 
这两个本质上是在抓取任务 走过去 然后抓取  

这个其实也不能算是全身控制 也是分层的 最后抓取的时候机器人底盘会被锁死 任务是走过去抓静态东西
视觉锁定 (Perception)： 腕部相机看向目标，SAM 给出 Mask，GSNet （iccv 之前的工作 可以 根据输入得到合适的抓取位置）持续计算出目标物体的最佳 6-DoF 抓取位姿。

高层决策 (Planning)： 高层策略持续拿到这个目标位姿（随着机器人走起来会更新的），查阅 GORM，得出机身应该移动到的最优 5D 目标状态（走到哪、蹲多低、倾斜多少度）。

底盘移动 (Locomotion - Tracking Phase)： 底层 RL 策略控制 12 个腿部电机，努力跟踪高层下发的 5D 指令。在此过程中，系统会限制机械臂带着相机在一个特定的“追踪球”内运动，（用传统控制 根据视角偏移量来控制）防止运动模糊丢死目标。（走的时候因为gsnet 持续工作 不能丢失视野）

停车抓取 (Manipulation - Grasping Phase)： 当底盘走到 GORM 允许的阈值范围内时（说明姿态已经完美），底盘停止移动（保持当前姿态锁死）。最后，机械臂通过精准的 IK 规划，伸出手完成最终的抓取

### GAMMA: Graspability-Aware Mobile MAnipulation Policy Learning based on Online Grasping Pose Fusion
ICRA2024. 

任务和上一个一样 也是走过去抓静态东西，只不过这里更关注 走过去的时候视角丢失   
在移动操作（Mobile Manipulation）中，机器人离目标越远，视野越开阔，但深度图越模糊；离目标越近，看细节越清楚，但由于视角变化和遮挡，很容易丢失目标的全局信息。
如果像传统方法那样，每帧画面都让 GSNet 独立预测一次抓取姿态，你会发现这些姿态会在空间中疯狂跳动（Flickering）。如果强化学习（RL）跟着这些跳动的目标走，机器人就会像喝醉了一样抽搐。

移动操作任务 (Mobile manipulation task)： 给定目标物体的位置 $p_{goal}$，机器人的任务是在未知环境中导航，有效地接近并抓取目标物体。我们遵循文献 [11] 和 [10] 中提出的主流设置。机器人配备了一个移动底座、一条机械臂和一个平行夹爪。机身安装有两个 RGB-D 相机：一个安装在移动底座的头部 ($D_{head}, I_{head}$)，另一个安装在夹爪上 ($D_{grip}, I_{grip}$)，其中 $d$ 和 $c$ 分别代表深度图像和彩色图像。机器人的移动底座在 $SE(3)$ 空间中采用 3 自由度 (DoF) 配置，并结合一个 $(x+1)$ 自由度的机械臂。具体而言，Spot 机械臂和 Unitree Z1 机械臂的 $x=6$，而 Fetch 机器人则为 $x=7$，此外还增加了一个用于物体抓取的 1 自由度夹爪。概述 (Overview)： 图 2 展示了我们提出的“可抓取性感知” (graspability-aware) 移动操作方法的概览。为了实现可抓取性感知，我们的方法处理夹爪深度图像 $I^d_{grip}$，并利用现成的抓取模块 GSNet [14] 来预测抓取位姿（第四节-A）。这些预测出的抓取位姿随后进行在线融合（第四节-B），并编码为可抓取性状态 $S_{grasp}$。随后，我们的方法通过强化学习 (RL)，结合视觉信息 $S_{visual}$ 和状态信息 $S_{state}$，学习可抓取性感知的移动操作策略 $\pi(A_{base}, A_{arm}, A_{grip} \mid S_{grasp}, S_{visual}, S_{state})$（第五节-A）。在我们的方法中，策略会生成：用于移动底座控制的 3 自由度 $SE(3)$ 速度 $A_{base}$、用于当前机械臂关节的 6 自由度残差调整 $A_{arm}$，以及用于控制夹爪的 1 自由度开关 $A_{grip}$


这里虽然有两个相机 但是确认目标位置 点云处理 可达性 只用了 机械臂的相机 本体相机 主要是喂给policy了 我的理解就是 点云处理 会得到一个点云候选空间 并随着行走 剔除 随着走动gsnet预测的不好的点 然后保留好的点 那就是抓取目标了 然后policy去学  最终有一个 复合奖励  由 点云总分数 和 夹爪到目标物体距离两部分构成 前者会促使机器人 以更好的视角 靠近 目标 后者抓取 随着时间 前者权重减小后者增加 

### Learning coordinated badminton skills for legged manipulators
ETH 2025  
这篇是很典型的whole-body visuomotor control：ANYmal-D + DynaArm 打羽毛球 任务不是简单走过去再操作 而是要同时做感知 预测 移动身体 挥拍  
和很多mobile manipulation不同 它不是上层给base速度/机械臂目标 然后底层分别跟踪 而是一个统一RL policy直接输出18个关节的position command 所以腿和手臂的协调是策略自己学出来的  

整体pipeline：ZED X双目相机检测羽毛球 -> 得到相机坐标下球位置 -> 结合机器人位姿变换到全局map frame -> EKF估计球状态 -> 羽毛球动力学模型预测轨迹和拦截点 -> 生成swing target position / swing time / racket orientation / swing velocity -> RL policy根据本体状态和这些目标输出全身关节命令  
部署时状态估计400Hz policy 100Hz 感知模块60Hz异步跑在Jetson AGX Orin上  

#### 仿真训练怎么做
在IsaacGym + legged_gym里训 用的是N-P3O 也就是带约束的PPO 因为机械臂电流有8A硬约束 普通soft penalty不够安全  
训练环境里包含比较细的机器人动力学：机械臂传动建模 actuator建模 系统辨识后的摩擦/阻尼/armature参数 还加了domain randomization 比如摩擦系数 base mass random push等  
为了让机器人不只是学一次性挥拍 一个episode里放了6个swing target 每个目标大约间隔2s 这样会学到击球后的follow-through和回到准备姿态  

actor-critic是非对称的  
actor只拿真实部署能拿到的信息：带噪本体状态 最近一次球方向 是否看到球 swing target swing time last action等  
critic可以拿privileged信息：无噪base/joint状态 EE状态 perfect shuttle perception 下一次挥拍目标 剩余挥拍次数等  
这里这么做的原因是多个击球目标会让value和“还剩几次击球/下一个目标在哪里”强相关 这些信息不给actor 但给critic能让value估计更稳  

球轨迹在仿真里不是用视频学习出来的 也不是从渲染图像检测出来的 而是先按随机初始位置/速度和空气动力参数预采样一批ground-truth轨迹  
大概从对方场地中心附近发球 初始位置/速度随机 然后用羽毛球空气阻力模型积分 得到会飞到机器人这边的完整真值轨迹  
动力学模型是 m dv/dt = mg - m ||v|| v / L 其中L是aerodynamic length 他们实测用4.1m  
每次训练采样一条球轨迹和目标击球高度 再平移/补齐轨迹 让球在指定swing time穿过目标高度  
所以仿真里是“先有真值轨迹” 但这个真值轨迹不会直接给actor 而是要经过模拟相机观测和EKF之后 再变成策略能用的目标/估计  

最关键的是感知噪声也放进训练loop  
他们先在真实机器人上采数据：相机围着一个已知位置的固定羽毛球运动 mocap提供相机位置 于是可以算相机到球的距离和相机角速度  
然后把检测概率和位置测量误差回归成 距离 + 相机角速度 + 是否在FOV 的函数  
训练时每个时间步会根据当前机器人相机姿态 判断真值球位置是否在FOV里  
如果在FOV里 就按这个noise model采样是否检测到球 以及3D位置测量误差 伪装成一帧真实相机观测  
这些连续的带噪position观测再喂给和真实部署完全一样的EKF + trajectory predictor  
这里要注意：一帧观测只有position 没有可靠velocity 速度是EKF根据连续多帧position观测 + 羽毛球动力学模型逐渐估出来的  
EKF每次更新position/velocity后 trajectory predictor都会重新往未来rollout一条轨迹 再更新当前预测的拦截点和swing time  
所以策略不是看到完美球状态 而是看到和真实相机相近的连续有噪估计 最终reward里还可以直接惩罚EKF预测拦截点和真实拦截点的误差  
这点很重要：它避免了teacher-student里teacher看完美状态导致不会学主动感知的问题 策略会自己学到pitch up多看一会球 再pitch down准备挥拍  

reward主要是time-based swing reward：只在击球时刻奖励EE位置 方向 速度跟踪 另外有perception error reward 还有torque/action rate/joint limit/collision/stand still等正则  
perception reward不是简单keep in FOV 而是最终拦截位置的估计误差 这比硬写FOV reward更合理 因为不会为了看球过度牺牲动作效率  

#### 真实部署怎么做
真实系统里羽毛球检测非常工程化 不是神经网络 也是持续检测/持续预测 不是看到一张图就一次性确定轨迹  
因为用的是橙色羽毛球 所以直接HSV颜色阈值过滤：Hue < 5 或 >176 Saturation >60 Value >160 这样把球从背景里分出来  
然后利用ZED X双目把2D检测点恢复成3D点 再用同步时间戳和机器人位姿变换到map frame  
他们强调这个map frame要用MSF + CompSLAM得到的全局一致坐标系 不能直接用会漂移的odometry frame 因为球状态是在这个frame里滤波 漂移会导致EKF和最后拦截点都错  
每一帧检测到的球3D位置进入EKF EKF参数和训练时一致 输出滤波后的球状态 position + velocity 然后轨迹预测模块算未来轨迹和拦截点  
单帧双目只能给当前3D位置 速度需要连续几帧后才能估稳 所以论文里说对手击球后平均约0.357s 感知模块才register出可用于拦截的轨迹  
预测轨迹会随着新观测不断更新 可以理解为持续输出一条带空气阻力的“抛物线” 但不是理想抛物线 而是羽毛球阻力模型下的未来轨迹  
如果球飞出FOV 系统会保留最后一次预测拦截点最多2s 继续尝试击球  
相机选择上他们反而没有用广角 而是选了较窄FOV来换更高角分辨率 因为高速小目标的角度测量噪声很关键 但这也导致后方/头顶球更容易丢视野  

硬件上还有一些部署细节：机械臂没有准确力矩测量 所以他们用CMA-ES在仿真里拟合机械臂模型参数 让同样command下的仿真关节轨迹匹配真实轨迹  
腿部用已有actuator network思路 机械臂因为是QDD且动力学更透明 所以用参数辨识方式  
部署时还用轨迹qualification heuristics：预测轨迹要同时穿过两个矩形区域 一个在网高1.55m附近 一个在地面服务区 才认为值得击球  

#### limitation / future work
作者明确说现在高层击球命令还是规则化的 swing height / velocity / orientation 都由一套可配置规则决定 没有学一个真正的高层badminton strategy policy  
future work可以根据对手身体动作/站位自适应选择击球命令 这样不只是把球打回去 而是更像真实比赛策略  
当前策略主要训练在相对固定的击球高度 约0.9-1.4m above base 且主要用同一面球拍击球 作者也说可以扩展更多挥拍动作和更多击球方式  
返回落在机器人身后的球成功率更低 主要限制来自感知：往后走时更难把球保持在FOV里 如果给ground truth perception 仿真里性能会更对称  
可能改进是更大FOV相机 或者加一个可控pitch的相机云台  
当前感知强依赖单个ZED双目 + HSV颜色阈值 + EKF 光照和背景颜色变化会改变噪声水平 所以不同场地的perception sim-to-real gap仍然存在  
作者提到可以加入更多模态 比如力矩/声音检测击球时刻 额外RGB/depth/event camera 提高高速交互下的响应  
另外人类会通过观察对手动作提前预测球路 所以human pose estimation也可以作为未来的输入模态  

我的理解：这篇最大的价值不是“羽毛球打得多好” 而是把真实感知噪声模型和部署同款EKF/trajectory predictor嵌进RL训练 让whole-body policy同时优化运动和感知质量  
这比单纯给机器人一个动态目标点去追踪更强 因为它学到的是为了未来能估计准 现在身体应该怎么动

### Learning Multi-Stage Pick-and-Place with a Legged Mobile Manipulator
RA-L 2025 Horizon Robotics  
这篇方法叫SLIM（Sim-to-Real Learning of Long-Horizon Legged Manipulation 作者自命名） 重点不是动态抓取 而是长时序mobile pick-and-place真实部署  
硬件是 Unitree Go1 + top-mounted WidowX-250S arm 腕部装 Intel RealSense D435 相机  
任务是语言指定颜色：Drop the {color1} cube into the {color2} basket 机器人要搜索目标cube 走过去抓起来 再搜索目标basket 搬运过去 放进去  
它最重要的特点是：完全在仿真训练 zero-shot真实部署 只用一个wrist-mounted camera 没有外部全局相机 没有假设目标一开始可见  
真实实验400个episode full task成功率约78.3% 平均完成时间43.8s 甚至比专家遥操作快  

#### 任务设置
对象是彩色cube和basket 颜色来自 red/blue/green/yellow 有distractors  
标准布局是机器人在2m*2m区域中心 物体/篮子在四角附近 训练时每个角落位置在半径0.5m圆内随机  
还有Cluttered布局 物体更靠近 更像OOD（out-of-distribution 分布外）测试  
任务分成多个stage：Search -> MoveTo -> Grasp -> SearchWithObj -> MoveToWithObj -> MoveGripperToWithObj -> DropInto  
这些stage label是训练teacher用的privileged subtask id 真实部署的student没有stage id输入  

#### 总体pipeline
分层控制：高层visuomotor policy + 低层locomotion policy  
低层policy先训练好并冻结 只负责四足运动 接收高层给的locomotion command 输出腿部控制  
高层policy负责长时序任务决策 处理视觉/语言/本体信息 输出两类控制信号：  
1）arm and gripper control signals 直接发给机械臂driver  
2）locomotion command 发给低层policy 控制Go1移动  
论文里高层动作 ahi 包括 locomotion command（目标前进速度和角速度） arm delta joint position 和 gripper control signal  
所以这篇不是统一policy直接输出所有关节 而是高层管任务和手臂增量/夹爪 低层管腿  
部署频率：高层10Hz 低层50Hz driver约500Hz 异步运行  

#### Low-level policy
低层policy只学locomotion 训练方法跟 Walk These Ways 类似 用RL训练跟踪采样的线速度/角速度命令  
训练完成后冻结 teacher和student都共用同一个低层policy  
这使高层可以主要学习search/grasp/drop这样的任务逻辑 不必同时从头学四足步态  
这里我再确认了一下原文：这篇明确写的low-level command是 sampled linear and angular velocity command 高层给的locomotion command也解释成target forward and angular velocities  
没有看到作者把base height / pitch作为高层显式输出命令来写 但因为低层follow的是Walk These Ways一类locomotion policy 具体底层内部可能还有gait/body相关输入或默认设定 论文这里没有展开  

#### Teacher怎么训练
teacher只在仿真中训练/运行 输入是privileged low-dimensional structured state  
明确包括：object positions / orientations / object categories / subtask id / robot proprioception 等  
这里作者做了一个很重要的限制：visibility mask 如果某个object不在wrist camera视野内 那它的object feature会被置零 只有相机FOV里可见物体的特征保留  
所以teacher虽然有物体真值 但不是完全全局上帝视角 它仍然被限制成“只能知道相机看见的物体” 这样更接近student的部分可观测设置  
teacher输出和student一样的高层动作：locomotion command + arm delta joint position + gripper control  

teacher的难点是long-horizon RL 容易出现两个问题：  
loss of capacity / lost plasticity：训练到后面网络学新stage变难  
catastrophic forgetting：学后面stage时忘掉前面已经学会的search/move/grasp  
还有continual exploration问题：达到一个milestone后还要继续探索下一个stage 否则会停在局部最优  

作者的解决方法是 Progressive PEX（Progressive Policy Expansion 渐进式策略扩展）  
PEX本身是Policy Expansion 作者借来的思想 这里做成按subtask逐步扩展  
具体做法：teacher不是一个大网络管所有stage 而是一组结构相同的子policy Π = {π1, π2, ..., πK}  
每个subtask k有一个独立policy πk subtask id作为gate 进入哪个stage就激活哪个policy  
训练时一开始只有π1学第一个stage 每遇到新stage就新增一个policy网络 旧stage的policy保留  
好处是：新stage有新的网络容量和探索能力 旧stage不容易被覆盖遗忘  
代价是：teacher依赖privileged subtask id 只能在仿真训练中用 student部署时不能用  
这里“怎么确认到了新阶段”是由仿真环境用privileged状态维护的 比如目标进入视野/接近cube/成功grasp/cube在夹爪中/看到basket/接近basket/cube进入basket等条件  
也就是说teacher的stage切换不是从图像里学出来的 而是仿真环境根据object pose gripper state success condition等直接给subtask id  

teacher reward也利用了任务分解  
每个stage都有sparse subtask success reward和distance-based shaping reward  
SearchWithObj / MoveToWithObj里还加了arm-retract reward 鼓励抓起物体后把手臂抬回一个canonical pose  
这个reward很关键：一方面让wrist camera获得更好的视角去找basket 另一方面避免机械臂贴地伸太远导致电机接近力矩极限和实物抖动  
另外还加了行为先验：stationary manipulation（操作时尽量站定）和rotational search（搜索时原地旋转 不走出workspace） 这些对安全和真实部署很重要  

#### Student怎么训练
student才是真实部署的visuomotor policy  
student不是多个actor 也不会像teacher那样每到新阶段激活新的πk 作者明确说final visuo-motor policy不使用subtask id  
所以student是一个单一视觉策略 它只能从wrist camera 本体状态和语言里隐式判断自己处于search/grasp/transport/drop哪个阶段  
输入 sstu = [o, sr, l]：  
o 是stacked ego-centric RGB images 也就是腕部相机RGB图像序列  
sr 是机器人proprioceptive state 本体状态  
l 是language instruction 语言指令 比如 Drop the red cube into the blue basket  
论文同时说为了降低visual sim-to-real gap 在perception module和policy之间加入visual information bottleneck：segmentation map + depth map  
所以可以理解为原始wrist RGB经过视觉模块/增强处理后 给policy更低域差的分割/深度表示 但整体仍然只依赖单个腕部相机  

student训练不是纯行为克隆 也不是直接从pixel RL  
他们修改SAC（Soft Actor-Critic 强化学习算法）做 distillation-guided RL  
replay数据用mixed rollout：每个episode开始时 选择整段episode由teacher rollout 或由student rollout  
teacher rollout提供高质量长时序轨迹 帮student覆盖后续stage 否则student早期根本走不到DropInto  
student rollout让student探索自己会遇到的状态分布  
目标函数里去掉SAC原本的entropy reward 用teacher-student KL distillation loss替代 并保留RL的Q优化  
为了让KL项不只是死板模仿 他们把teacher action distribution的mode保持不变 但设一个固定dispersion/std 让student既学teacher mode又保留探索  
从实现角度看 不是单独做一个MSE supervised learning就结束 而是在student actor update里加一个distillation/KL项 同时还有critic/Q learning的RL loss  
如果actor是Gaussian policy 固定方差时KL大致会像约束student mean接近teacher action/mode 但SLIM保留了RL reward 所以student还能根据任务成功反馈修正teacher模仿误差  
这点和DQ-Net不同：DQ-Net student基本是teacher action MSE蒸馏 没有让student继续通过RL reward试错优化 所以更容易有teacher-student gap  
对比实验里 Distillation Only 只蒸馏没有RL loss 成功率和稳定性都更差 No Distillation 从视觉直接RL完全学不会  

#### Sim-to-real怎么做
动力学gap：  
1）机械臂用PID控制降低tracking error  
2）arm control perturbation 给arm joint position target加随机噪声  
3）arm mount perturbation 每个episode随机扰动机械臂安装位置和yaw 模拟真实安装误差/躯干高度变化  
4）object perturbation 让策略对物体相对位置变化更reactive 不要记死固定抓取轨迹  

视觉gap：  
1）visual information bottleneck：segmentation + depth map 降低像素域差并提高可解释性  
2）color modeling and randomization：用真实采样的HSV颜色作为seed 在仿真里随机颜色  
3）visual augmentation：随机纹理 背景物体 pixel augmentation spatial augmentation  
No Visual Aug在仿真里看起来成功率高 但真实会被背景误检干扰 尤其找basket掉得明显  

#### 观测和输出总结
Teacher输入：robot proprioception + visible object privileged states（物体位置/姿态/类别 但FOV外置零）+ subtask id  
Teacher输出：locomotion command + arm delta joint position + gripper control  
Student输入：wrist camera视觉序列 + robot proprioception + language instruction 实际部署只用机载腕部相机和本体传感器  
Student输出：同teacher的高层动作  
Low-level输入：高层给的前进/转向速度命令  
Low-level输出：腿部控制命令 负责四足移动  
Arm/gripper：高层动作直接经机械臂driver/PID执行 arm是delta joint position gripper是开合控制  

#### 实验和结果
主实验在真实Lobby标准布局下 每种方法3个seed 每个seed 20 episodes  
SLIM full task成功率78.3±5.8% Grasp阶段96.7% Search+MoveTo(WObj)阶段96.7% 平均43.8s完成  
Human Teleop是75.0% full task 平均65.5s 所以SLIM在这个设置下略高于人类遥操作 且更快  
No Arm Retract最后DropInto只有5% 因为抓起后手臂低垂 相机视角差 机械臂抖 容易掉物体  
No Perturbation抓取成功率低 因为策略在仿真里记住固定抓取轨迹 实物里位置稍有偏差就够不到  
No Visual Aug找篮子时容易被背景误检干扰  
Distillation Only能有一定表现 但方差大且不如distillation-guided RL  
No Distillation直接视觉RL在仿真里都学不起来  

泛化实验：同一个SLIM policy在Outdoor Carpet Room Kitchen Lobby-Cluttered等真实场景测试 成功率接近主实验 说明颜色/背景/地面变化下有一定泛化  
还展示了novel object shape re-grasping human interruption task chaining等emergent behavior  

#### failure modes / limitation / future work
失败模式：  
抓取时四足停晚了 把cube踢走  
低层policy抖动或后退 导致cube刚好超出机械臂可达范围  
DropInto失败 有时太早松手掉在basket外 有时后退靠近basket时把basket踢走  
少数情况下夹爪只夹住cube上部 cube挡住wrist camera 机器人在篮子上方悬停但不松手  

作者future work：  
提升locomotion policy训练 让底层移动更稳更适合移动操作  
增加视觉和语言多样性  
扩展到更多物体和更多任务  

我的理解：这篇和DQ-Net/羽毛球都不一样  
DQ-Net偏CV/grasp representation 但没实物；羽毛球偏动态轨迹预测和active perception；SLIM偏长时序真实移动操作  
它的价值在于证明了只用腕部相机 + 仿真训练 + teacher-student + progressive policy expansion 可以把多阶段pick-and-place做到真实约80%  
但任务物体还是彩色cube/basket 视觉语义相对简单 抓取也不是复杂6D姿态抓取 更像长时序移动操作系统验证  

### Learning Dynamic Pick-and-Place for a Legged Manipulator
RA-L 2026 KAIST / HKUST(GZ)  
这篇和SLIM都是pick-and-place 但重点几乎相反：SLIM强调长时序视觉搜索和真实sim-to-real 这篇强调高动态whole-body控制 重物 以及大高度差  
它没有解决复杂视觉感知 真实部署里用Vicon motion-capture system提供object和table pose 所以更像是“假设目标状态已知以后 怎么让腿臂机器人快速连续地抓-搬-放”  
硬件是自研四足平台 + Unitree Z1 6-DoF机械臂 + 1-DoF gripper 有效payload约2.3kg  
真实实验平均成功率73.3% 平均执行时间4.06s 物体最大1.3kg 高度范围从地面到1.1m桌面  

#### 任务设置
任务是dynamic pick-and-place：机器人在不暂停/不重新抓取/不慢慢修正的情况下 连续完成 grasp -> transport -> place -> retreat  
环境里有两个半径10cm的圆桌 一个pick table 一个place table 物体是直立放置的cylinder或cuboid  
每个episode随机pick table相对机器人位置 place table相对pick table的位置和高度 物体大小/质量/类型也随机  
训练范围：pick distance 0.9-3.0m pick height 0-1.3m place distance 0.5-3.0m place height 0-1.3m object mass 0.2-2.3kg  
成功标准：物体直立放到place table中心5cm内 并且EE retreat至少10cm 不扰动物体 10s内完成  

#### 总体pipeline
方法是hierarchical reinforcement learning 分两级：low-level locomotion controller + high-level pick-and-place controller  
Step 1 先训练低层locomotion controller 让它在随机机械臂扰动下保持稳定 并跟踪base/body命令  
Step 2 冻结低层actor 训练高层pick-and-place controller 高层同时输出base命令和arm/gripper命令  
这篇不是像SLIM那样把手臂delta joint交给外部arm driver 也不是像DQ-Net那种EE increment + IK 而是高层直接输出6个arm desired joint angles + gripper开合  
最终执行：低层输出12个腿关节目标角 高层输出6个arm关节目标和gripper binary command 所有关节目标通过500Hz PD controller变成torque  
部署频率：high-level 50Hz low-level 100Hz PD 500Hz  

#### Low-level controller
低层生成12个腿关节desired joint positions 用来做whole-body locomotion和body stabilization  
低层输入命令包括 base velocity command [vx, vy, wz] 和 body-control command [∆h, ϕ]  
这里∆h是base height offset ϕ是base pitch angle 这正好是之前SLIM里没有明确写出来的部分 这篇明确把height/pitch作为base posture command  
低层观测 o_low = [x, d, a]：x是base+leg proprioceptive state d是arm proprioceptive state a是previous action  
低层critic是asymmetric actor-critic 有privileged obs：base height base linear velocity 每条腿air/stance time ground reaction force foot周围height-scan等  

低层训练的关键是Random Arm Motion Generator  
它不是只用简单EE直线插值来扰动手臂 而是在joint-space里给每个arm joint随机初始位置/目标位置/轨迹时长  
每个joint独立选择constant-velocity或symmetric constant-acceleration模式 所以会产生异步的复杂机械臂运动  
训练时还会在end-effector挂0-2kg随机payload 用来模拟抓物体时手臂对base的扰动  
这样低层学到的是“机械臂乱动/带重物时也稳住base” 而不是只适配一类平滑EE轨迹  
实验里这个低层比VBC式EE线性插值扰动训练更稳 真实用baseline低层时nominal任务从仿真80%掉到真实30% 主要因为arm motion把base搞不稳甚至摔倒  

#### High-level controller
这篇不是teacher-student蒸馏 也不是多个actor按阶段切换  
它主要是两个actor：low-level actor先训好冻结 high-level actor再用PPO直接训练 pick-and-place任务  
high-level是单个actor 只是观测里显式输入了i_phase 所以策略靠phase index知道当前大概处于哪个任务阶段  

高层动作 a_high = [a_base, a_arm]  
a_base = [vx, vy, wz, ∆h, ϕ] 其中vx范围 -1到2m/s vy范围±1m/s wz范围±1rad/s pitch ϕ范围±0.28rad ∆h范围-0.2到0m  
a_arm = [q_des_arm0 ... q_des_arm5, a_gripper] 也就是6个机械臂目标关节角 + gripper binary open/close  
所以高层同时决定“身体怎么跑/蹲/俯仰”和“手臂每个关节怎么动”  

高层观测 o_high = [s_r, s_o, a_high_prev, p_pick, p_place, i_phase]  
s_r 是robot proprioceptive state  
s_o 是object state  
a_high_prev 是上一时刻高层动作 给短时行为上下文  
p_pick / p_place 是pick位置和place table位置 都在robot coordinate frame里  
i_phase 是当前任务阶段index 帮助长时序任务  
[s_r, s_o, a_high_prev] 还会作为history buffer输入  

object state不是完整SE(3) 而是keypoint representation  
因为物体假设一直直立 所以用3个关键点表示：object center top bottom 还加object size和shape one-hot（cuboid/cylinder）  
这种表示不包含yaw等完整6D pose 但是对直立pick-and-place足够稳定  
注意真实部署时这些object/table state来自Vicon 不是机载视觉 这是这篇最大现实限制  

阶段index i_phase有6个状态：  
0 initial state  
1 end-effector above pick table  
2 end-effector above place table  
3 object placed on place table  
4 object released  
5 arm returned to initial posture  
作者明确说没有用strict grasping condition作为phase transition 因为经验上不加严格抓取条件整体成功率更高  
i_phase本质上是外部任务状态机给的 需要知道EE/object/table之间的几何关系 比如EE是否到pick table上方 object是否放到place table上 gripper是否释放 arm是否回到初始姿态  
所以真实部署时不仅object/table state依赖Vicon 连phase切换也高度依赖这些外部状态估计 如果没有动捕 就必须用机载视觉估计object/table pose和任务状态 否则high-level actor的关键输入不完整  

#### Online mass estimation
这篇很重要的点是online mass-adaptation module 在线质量估计模块  
因为任务要抓0.2-2.3kg的物体 接近Z1有效payload上限 物体质量会显著影响提起高度/姿态/稳定性  
机器人没有直接测质量的传感器 所以作者把object mass m_obj 和gripper-object contact state C_contact定义成privileged observation  
训练时用仿真真值监督一个LSTM estimator 根据交互历史 [s_r, s_o, a_high] 估计质量和接触状态  
估计结果再拼回actor observation 给高层policy用  
真实中质量估计会在抓起后短暂震荡 然后在搬运过程中收敛到真实质量附近  
和latent adaptation相比 显式估计低维物理量训练更高效 作者说每iteration训练时间降低约11.32% 性能接近  
w/o estimation成功率低约4% 物体质量越偏离训练均值 gap越明显 重物时尤其需要估计质量来主动增加lift margin  

#### Curriculum和reward
高层训练用success-rate-driven curriculum 成功率驱动课程学习  
三个课程变量 L_pick L_place L_release 从0.10开始 根据对应subgoal success rate每50 iter增加0.01 逐步推到1.0  
课程难度控制：机器人到pick table距离 table间距离 object mass place/release成功标准  
为了避免只会pick但很少练place 作者加了同步约束：L_pick和L_place必须在一定margin内共同推进  
阈值：pick 0.90 place 0.85 release 0.85 regulation margin 0.015  

reward按六个stage组织：pre-grasping grasping carrying placement retreating finishing  
pre-grasping有EE to Obj EE Obj Contact  
grasping有Grasping Success  
carrying有Base Heading Obj to Place Base to Place  
placement有Place Success Gripper Release  
retreating有Base Retreat EE Retreat  
finishing有Complete  
另外有arm penalty base penalty manipulation penalty 控制关节速度/力矩/action变化/base motion/过大接触力/物体倾倒等  
作者强调每个stage只激活当前subgoal相关reward 前面stage完成后对应reward不再active 避免长时序value估计混乱  

#### 实验结果
仿真中ours成功率86.05% 平均2.071s  
Latent Adaptation 85.82% w/o Estimation 81.88% Segmented Policy 78.40%  
segmented policy把approach-and-grasp和transport-and-place分开训练 执行更慢且成功率低 因为grasp阶段只关注抓住 没有考虑抓完后的pose是否适合place 容易短视  
动态抓取时 base在gripper闭合瞬间仍有约1m/s平面速度和-1rad/s yaw rate 但EE速度降到约0.2m/s 说明策略学会了base继续运动但末端相对物体减速  

真实实验6个scenario 每个10次：Nominal Heavy Object Light Object Square Object Large Size Large Height Gap  
总成功率73.3% 平均4.06s  
Nominal 9/10 Heavy 8/10 Light 8/10 Square 6/10 Large Size 7/10 Large Height Gap 6/10  
Large Height Gap是地面到1.1m桌面 需要base pitching和arm extension 平均5.20s 成功率60%  
真实最大只测到1.3kg 虽然训练到2.3kg 因为gripper safety mode限制了可用夹持力  

#### failure modes / limitation / future work
多数失败发生在grasping  
EE通常能到可抓位置 但手指闭合时物体会滑出或被弹出  
一旦稳定抓住 后续transport/place/retraction通常能完成  
Square Object现实只有60% 主要因为仿真gripper简化成cuboid 接触模型低估了棱角接触 对方柱体抓取影响很大  
gripper只有binary open-close position command 不能根据物体形状/摩擦调节抓力  
Large Height Gap里的近地抓取接近平台运动学极限 可行抓取角度少 whole-body stabilization更敏感  
最大限制：真实部署依赖Vicon外部动捕来提供object/table pose 不是机载视觉闭环  
作者future work明确说：去掉外部motion-capture 引入ego-centric vision 并扩展到更多object geometries和physical properties  

我的理解：这篇可以看作SLIM的互补  
SLIM解决“只靠腕部视觉的长时序搜索-抓取-放置” 但动作慢 物体简单 高度/重物不强  
这篇解决“目标状态已知时 怎么做高速连续whole-body pick-and-place” 但视觉感知被Vicon替代  
对我们想做真实动态抓取来说 这篇最有价值的是：低层随机机械臂扰动训练 + 高层显式控制base height/pitch + 在线质量/接触估计 + 统一高层策略优于分段策略  
但如果要变成真实自主系统 还需要把Vicon object/table state换成机载视觉估计 这正好是和DQ-Net/SLIM可以结合的地方  
