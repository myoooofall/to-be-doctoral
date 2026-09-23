# Underwater Robot Learning-based Navigation / Motion Control

这里先明确focus：我们主要看水下机器人的learning-based导航和运动控制  
不重点看水下建图/SLAM/三维重建 本文里如果出现state estimation/visual odometry 只把它当成训练标签生成或控制闭环需要的辅助模块  

水下机器人和地面/腿式机器人的不同点：  
GPS不可用 视觉容易受浑浊/光照/水流影响 低层动力学里浮力/阻力/水流非常重要  
所以水下learning方向大概分两条：  
1）视觉/传感器输入到导航动作 比如mapless navigation goal-conditioned navigation  
2）状态/目标输入到低层控制 比如6-DoF position control docking trajectory tracking  

## 给四足RL研究者的水下控制最小知识框架

本文默认读者已经熟悉四足机器人的RL感知、locomotion control和navigation，但不要求先系统学习传统船舶控制或计算流体力学。目标是掌握足以正确设计observation、action、simulator、domain randomization和真实实验的最小水下背景。  

从四足RL迁移到水下RL，可以先建立下面的对应关系：  

| 四足机器人 | 水下机器人 |
|---|---|
| base pose / velocity | AUV的6-DoF position、orientation、linear/angular velocity |
| joint torque / joint position action | individual thruster PWM，或body-frame 6D wrench再做thrust allocation |
| 重力、接触力、摩擦 | 重力、浮力、drag、added mass、current与推进器力 |
| 地形变化、负载变化 | payload、CoM/CoB变化、水动力参数、海流和推进器差异 |
| IMU、joint encoder、foot contact | IMU、DVL、depth sensor、USBL、camera / sonar |
| GPS / lidar map localization | 水下通常无GPS，依赖DVL+IMU+depth、USBL、视觉/声学相对定位 |
| locomotion policy + navigation policy | low-level pose/velocity controller + guidance/path planner |

### 必须理解的五件事
1）Reference frame和6-DoF state  
至少能区分world/NED frame与body frame，知道surge/sway/heave和roll/pitch/yaw分别表示什么；不需要从头推导完整坐标变换理论。  

2）相对水速而不是只看对地速度  
水动力drag物理上取决于v_relative = v_robot - v_water。训练中常把水设为静止，于是v_relative等于机器人body velocity；真实海流存在时两者不同。  

3）执行器接口  
direct-to-thruster policy更接近四足直接输出joint command，能学习推进器耦合但绑定具体硬件；body-wrench policy类似先输出期望base wrench，再由传统allocation分配到推进器，更通用但依赖分配模型。  

4）水下状态估计不是默认可得  
IMU给姿态/角速度，depth sensor给深度，DVL主要给相对海底或水体的速度；绝对/水平位置还需要积分、USBL、视觉marker、声学定位或外部动捕。论文把position error放进observation，不代表机器人天然知道自己位置。  

5）sim-to-real gap既来自参数，也来自缺失的物理结构  
随机化mass、volume、CoB-CoM offset和thruster gain只能覆盖模型中已经存在的变化；如果仿真器完全缺少cross-axis drag、current、motor delay或sensor dropout，单纯扩大DR不一定能补回来。CORAL-AUV正是在研究这个问题。  

### 当前可以暂时不深挖的内容
如果研究目标是水下RL控制，而不是传统控制理论或CFD本身，目前可以先不花大量时间在：  
Navier-Stokes方程的严格推导与数值格式  
完整Fossen 6-DoF矩阵每一项的手工推导  
滑模、自适应控制和非线性控制的稳定性证明  
CFD网格生成、湍流模型和求解器开发  
精确水动力系数辨识的全部实验方法  

但需要知道这些模块在系统里做什么、有哪些假设、输出什么，以及它们的误差会怎样影响policy。对于RL研究，优先级应是：  
task与系统分层 -> observation/action接口 -> 传感器可获得性 -> simulator physics -> domain randomization/history/adaptation -> 真实闭环验证。  

### 推荐的RL研究切入层次
低层控制：desired pose / velocity -> body wrench或thrusters；重点是鲁棒性、能耗、故障和sim-to-real。  
高层导航：camera/sonar/range/goal -> waypoint或desired velocity；重点是partial observation、障碍感知和搜索。  
完整系统：perception/navigation policy输出目标，Sim2Swim或其他low-level controller稳定执行；不要把“输入相对目标点”误称为已经解决真实水下导航。  

## 论文时间线总表

表中的“任务”按机器人真正学习和执行的接口来写，不把作者笼统使用的navigation / end-to-end / agile等词直接当作能力结论。  
“有实物”也不等于完整任务已在真实水下闭环验证；若实物只验证了拆分后的子任务，会在表中明确标出。  

| 时间 | 论文 | 发表/状态 | 一句话任务与证据 |
|---|---|---|---|
| 2020 | Vision-Based Goal-Conditioned Policies for Underwater Navigation in the Presence of Obstacles | RSS 2020 | 视觉 + 相对目标输入，输出离散yaw/pitch进行近珊瑚反应式导航；有加勒比海真实实验，但恒速、无地图且依赖相对定位。 |
| 2022 | Mapless Navigation of a Hybrid Aerial Underwater Vehicle with DRL Through Environmental Generalization | LARS 2022 | 空水两栖机器人根据range和相对目标做mapless goal reaching及跨介质运动；Gazebo仿真，无实物。 |
| 2022 | Deep Reinforcement Learning for Adaptive Path Planning and Control of an AUV | Applied Ocean Research 129, 103326 | REMUS-100在二维未知场景中到随机目标并避静态圆障碍；TD3只输出方向舵、速度基本固定，使用6-DoF动力学仿真，无实物。 |
| 2023 | Reinforcement Learning for AUVs via Data-Informed Domain Randomization | Applied Sciences 13(5), 3133 | 4-DoF位姿调节；SAC给虚拟控制命令，再用真实数据学习action adapter / thrust mapping；有水池和动捕实物验证。 |
| 2023 | Dynamic Robotic Tracking of Underwater Targets Using Reinforcement Learning | Science Robotics | 水面Wave Glider用range-only声学测距估计水下目标，再由SAC选择高层waypoint；有Monterey Bay超过15小时海试，不是AUV底层控制。 |
| 2023/2024 | Adaptive Formation Motion Planning and Control of AUVs Using DRL | IEEE Journal of Oceanic Engineering | 3艘AUV二维定深编队、到目标和静态避障；TD3输出推进力与舵角，3-DoF简化仿真，无实物。 |
| 2024-03 | Path-Following Control of UUV Based on an Improved TD3 DRL | IEEE Transactions on Control Systems Technology 32(5) | **4-DoF given-path tracking**：仿真中同时跟踪x/y/z变化的真实3D螺旋；实物因水下RTK不可用，只分别验证水面二维路径和水下定深；混合真实replay重训，不是zero-shot。 |
| 2024/2025 | Toward 6-DOF AUV Energy-Aware Position Control Based on DRL | IEEE/OES AUV 2024报告；arXiv 2025 | Swim4Real前期版本：TQC直接输出8推进器命令做6-DoF定点位姿调节并惩罚动作能耗；仅仿真。 |
| 2024/2025 | Learning to Swim: RL for 6-DOF Control of Thruster-Driven AUVs | ICRA 2025 | 6-DoF目标位姿调节，PPO直接输出6个推进器；Isaac Lab大规模并行和domain randomization，zero-shot水池实物。 |
| 2025 | Swim4Real: DRL-Based Energy-Efficient and Agile 6-DOF Control for Underwater Vehicles | IEEE Robotics and Automation Letters 10(7) | **6-DoF固定目标位姿调节，不是轨迹跟踪**；TQC直接输出MOLA的8个PWM，Stonefish仿真训练约15小时，直接下水槽做10组实验；以PWM拟合功率估计约39%节能。 |
| 2025 | MarineGym: A High-Performance RL Platform for Underwater Robotics | IROS 2025 | GPU并行水动力仿真和控制benchmark平台，覆盖定点、轨迹和着陆；它是平台论文，不是一套完整导航policy。 |
| 2025 | Fast Policy Learning for 6-DOF Position Control of Underwater Vehicles | preprint / conference-stage work | 快速训练的6-DoF目标位姿控制与算法比较，比learning2swim多了一个轨迹跟踪，但是本质任务其实差不多，训练速度也差不多，有受控水池验证，仍非规划或避障。 |
| 2025/2026 | Sim2Swim: Zero-Shot Velocity Control for Agile AUV Maneuvering in 3 Minutes | conference-stage paper | 跟踪时变三轴线速度和任意姿态，policy输出6-DoF wrench并交给传统推进器分配；3分钟训练、zero-shot水池路径实验。 |
| 2025/2026 | Deep Reinforcement Learning for Autonomous Underwater Navigation: A Comparative Study with DWA and Digital Twin Validation（预印本题名：Digital Twin–Supervised RL Framework...） | Sensors 26(7), 2179 | **定深二维目标/gate导航 + 虚拟静态障碍避让**；PPO输出7档离散航向增量，训练采用二维运动学模型；海试用USBL和数字孪生提供位置、虚拟障碍及边界信息，BlueROV2真机执行，但没有真实声呐/相机障碍感知。 |
| 2026 | Sim-to-Reality Adaptation for DRL Applied to Underwater Docking | arXiv preprint；稿件标注under review by IROS 2026 | **3D相对位置 + yaw对准 + 接触式垂直入坞**；PPO根据3DBM/USBL提供的相对位姿输出6维body wrench，Stonefish训练约3小时；Girona AUV水池10次成功8次。所谓adaptation实际是噪声/随机初始状态/高保真接触建模后的zero-shot transfer，没有真实数据微调或在线适应。 |
| 2026-06 | Towards End to End Motion Planning and Execution for AUVs Using RL | arXiv v1 preprint，未确认正式收录 | **4-DoF goal-conditioned局部导航 + 静态避障**；高层以单目RGB、三帧FLS和oracle本体/目标状态输出相对(x,y,z,yaw)子目标，低层SAC再输出surge/sway/heave/yaw控制量并固定分配到8推进器。只在HoloOcean三个基础几何体场景中仿真；未做实物、真实定位或真实声呐，未见曲面/视角测试成功率最低2/20。 |
| 2026-07 | CORAL-AUV: CFD Oriented Reinforcement Learning for Autonomous Underwater Vehicles | arXiv preprint；MIT-WHOI / WHOI / MIT | **6-DoF目标pose/waypoint控制，任务框架直接继承Learning to Swim**；核心不是新PPO，而是用OpenFOAM CFD数据训练surrogate drag model并与惯性盒、真实coast-down System ID对比。CUREE水池和真实珊瑚礁外场实验表明CFD policy误差、时间和PWM平方和更低；尾部增加2 lb后只有CFD+DR policy完成全部任务。 |

### Vision-Based Goal-Conditioned Policies for Underwater Navigation in the Presence of Obstacles
2020 McGill / Toronto  
发表/收录：RSS 2020（Robotics: Science and Systems）  
方法名叫Nav2Goal 作者自命名 目标是在没有先验地图的情况下 让AUV（Autonomous Underwater Vehicle 自主水下机器人）根据用户给的相对waypoint导航 同时避障并尽量经过有科学价值的coral区域  
这篇不是建图论文 虽然用了visual odometry / EKF做相对定位 但核心是goal-conditioned imitation learning 学一个视觉反应式导航policy  

#### 任务设定
场景是珊瑚礁近距离巡检 机器人需要在离珊瑚很近的高度运行 既要避障 又要优先拍到珊瑚 而不是走最短路径穿过沙地  
用户给一串2D relative goals / waypoints 机器人依次接近这些waypoint  
机器人运动接近flat swimming：roll保持0 pitch目标大多围绕0 但policy仍然输出pitch动作来避障/调整高度  
它没有全局地图 也没有规划完整环境 主要是当前图像 + 相对目标 -> 当前转向动作  

#### 三阶段pipeline
第一阶段 Behavioural Cloning 行为克隆  
人工专家给机器人历史视频帧标注动作：当前图像应该怎么改yaw和pitch  
标注目标不是单纯去goal 因为这时还没有goal conditioning 而是“安全 + 科学采样”：避障 在没有障碍时把珊瑚放到视野中心  
用这些image-action pairs训练一个非goal-conditioned behaviour policy  

第二阶段 On-policy data collection + hindsight relabelling  
把第一阶段的behaviour policy放到真实机器人上跑 收集trajectory  
用机器人自己的relative localization / state estimation记录每个时刻的pose  
对轨迹中的任意时刻t 随机选未来某个t+∆t 把从xt到x(t+∆t)的相对位移当成goal  
于是原来的动作标签不用人工重新标注 就变成了训练样本 (current image, relative goal) -> executed yaw/pitch action  
这就是hindsight relabelling：机器人实际走到哪里 就把那里反过来当作当时的目标  

第三阶段 Deployment  
用户给一串relative waypoint 当前waypoint作为goal输入policy  
policy根据当前front image和relative goal输出yaw/pitch steering action  
如果机器人到达waypoint阈值范围 就切到下一个waypoint  

#### 网络/观测/输出
输入：单帧forward-facing RGB image + 2D relative goal  
goal用机器人当前坐标系下的平面坐标表示 论文比较了Cartesian (x,y)和polar (magnitude,yaw) 最后Cartesian更好  
网络：ResNet-18 CNN提取图像特征 goal先过dense layer 再和CNN feature结合  
结合方式比较了concat和multiplication 最后multiplication对goal更敏感 concat容易忽略goal  
输出：yaw和pitch两个categorical distributions 每个都是7类 C={-3,-2,-1,0,1,2,3}  
这些类代表相对当前图像的离散航向变化 yaw负/正对应顺/逆时针 pitch负/正对应向下/向上  
低层控制器再把离散yaw/pitch action转换成实际机器人执行命令 并做temporal smoothing减少动作抖动  

这篇动作空间很小 不是直接输出thruster 也不是6-DoF控制  
它更像视觉高层反应式导航policy：看图和目标 决定此刻往哪转/抬头低头  

#### 训练细节
Behaviour policy训练：ResNet-18 单图输入 两个softmax head分别预测yaw/pitch 7分类  
loss是cross entropy + label smoothing + Concrete Dropout的KL regularization  
Concrete Dropout还用于uncertainty estimation 推理时dropout保持开启 让OOD输入时动作有一定随机性  

Goal-conditioned训练：仍然是同样分类loss 只是多了goal输入  
训练数据来自trajectory hindsight relabelling 格式是 <image, goal, yaw action, pitch action>  
为了避免目标太远 采样未来timestep时设置最大时间差τ  

#### State estimation在这里的作用
这篇会用state estimation 但目的不是建图  
它有两个用途：  
1）训练时给hindsight relabelling提供相对pose 从而自动生成goal标签  
2）部署时判断机器人相对当前waypoint的位置 以及什么时候切换到下一个waypoint  

state estimator用downward-facing rear camera跑DSO（Direct Sparse Odometry） 因为front camera常看开阔水体 特征少 downward camera看珊瑚 特征更丰富  
DSO本身有尺度漂移 所以用downward sonar估计尺度 再和IMU depth sensor magnetometer constant speed prior一起进EKF  
这部分对我们来说不是核心 但说明水下无GPS时 即使做learning navigation 也通常需要某种相对定位来给goal和切换waypoint服务  

#### 探索策略
behaviour policy如果只稳定沿珊瑚走 数据会很单一  
作者用action distribution entropy作为uncertainty 在不确定高的地方混入exploration action  
具体做法是每隔一段时间采样一个探索yaw action 并commit一段固定时间 再用entropy作为gating weight把policy action和exploration action混合  
这样能增加轨迹多样性 同时尽量不破坏避障/看珊瑚的基本行为  

#### 实验
仿真：Unreal Engine水下环境 随机放coral 自动专家通过地图真值找最近可见coral 用PID生成expert action  
仿真收集约18000个 image/pose/action samples 6Hz 用hindsight relabelling训练goal-conditioned policy  
仿真结果说明 goal-conditioned policy不是直线去goal 而是会在去goal过程中尽量让coral保持可见  

真实海试：先用14000张真实图训练behaviour policy 再部署收集约20500个goal-conditioned训练样本  
真实部署在Caribbean Sea 有浪涌 光照变化 可见度变化  
最终跑4条10-waypoint trajectory：1条两次完整完成 1条到8/10 waypoint 另外两条到7/10 waypoint 因浪涌提前停止  
论文总称海试近1km autonomous visual navigation 到达约40个waypoints  

#### limitation
policy是reactive 只看当前图像和当前goal 没有记忆/地图 所以遇到需要大转弯的waypoint序列时可能overshoot  
机器人恒定前进速度 转弯半径受水动力/阻力限制 不是所有waypoint序列都可执行  
因此他们甚至用已执行trajectory segment拼接waypoint路径 保证任务在机器人能力分布内  
动作输出只是yaw/pitch分类 不是底层运动控制 更不涉及thruster-level控制  
goal是2D relative goal 水下三维目标/高度变化没有充分处理  
真实环境中浪涌和能见度变化明显影响控制 这些外部力没有被显式建模  

#### 对我们的启发
这篇很早 但有几个点值得记：  
1）水下learning navigation可以先不建图 走reactive policy路线  
2）如果要goal-conditioned 又不想人工标注goal动作 可以用hindsight relabelling从真实trajectory自动生成训练标签  
3）把“任务偏好”编码进行为克隆标签很直接 比如这里是避障 + 看珊瑚 而不是最短路  
4）水下系统即使不做建图 仍然需要相对定位/状态估计来定义goal和判断是否到达  
5）它是导航高层policy 不是运动控制policy 如果我们研究水下运动控制 还需要看6-DoF RL控制类工作  

### 按时间排序的相关工作

以下条目按论文首次公开或online publication时间从早到晚排列 正式卷期年份不同时会在条目中单独注明  
同一年但缺少明确公开月份的preprint只按大致时间放置 不把排列误解成严格的月度优先级  

#### Mapless Navigation of a Hybrid Aerial Underwater Vehicle with Deep Reinforcement Learning Through Environmental Generalization
2022 HUAUV  
发表/收录：LARS 2022（Latin American Robotics Symposium）  
HUAUV是Hybrid Unmanned Aerial Underwater Vehicle 空中-水下混合无人机器人/空水两栖机器人  
这篇做的是HUAUV的mapless navigation + medium transition  
medium transition指air-to-water和water-to-air介质切换 也就是机器人从空中进入水下 或者从水下飞出水面  
它不是纯水下AUV论文 但与后文的Digital Twin工作相比 更接近“传感器range输入 + RL导航策略”  

#### 任务和方法能力
任务是goal-oriented mapless navigation：给机器人一个目标位置 机器人只用距离传感器和相对目标信息导航到目标 同时避障  
同时还要完成介质切换：  
Air-to-Water (A-W)：从空中起飞/飞行 进入水中 再到水下目标  
Water-to-Air (W-A)：从水下出发 穿过水面 到空中目标  

这篇的核心贡献不是新的水动力控制器 而是比较不同Deep-RL actor-critic结构在HUAUV导航/介质切换中的泛化能力  
作者提出两类double-critic方法：  
TD3（Twin Delayed Deep Deterministic Policy Gradient 双延迟深度确定性策略梯度）确定性策略  
SAC（Soft Actor-Critic 软演员-评论家）随机策略  
并且把actor和critic都换成RNN/LSTM结构 用历史信息提高部分可观测场景下的稳定性  

#### Pipeline
1）在Gazebo + ROS里搭建HUAUV仿真  
空中部分用RotorS框架 支持多旋翼控制、风扰动等  
水下部分用UUV Simulator 支持hydrostatic/hydrodynamic effects thrusters sensors external perturbations  

2）建立两个水箱环境  
环境1：10x10x6m walled tank 一米水柱 四个圆柱障碍 类似subsea drilling risers  
环境2：更复杂的sea platform结构 包括墙、半墙、管道等  
训练主要在环境1 做泛化测试时直接跑到未见过的环境2  

3）构造range + relative goal observation  
空中用模拟LIDAR 水下用模拟FLS sonar  
policy不输入地图 不输入图像 不输入显式全局位置  
但输入里有relative localization to target 也就是机器人相对目标的位置/角度  

4）训练TD3/SAC double-critic RNN policy  
训练1500 episodes 每个episode最多500 steps  
目标点随机生成 到达目标、碰撞或超时结束  

5）测试跨环境泛化  
用在环境1训练的policy测试环境1和未见过的环境2  
再比较只在环境2训练、环境1+环境2继续训练等设置  

#### 仿真器 / 是否开源
仿真器用的是Gazebo + ROS + RotorS + UUV Simulator  
这些基础框架本身都是常见开源工具  
论文说“A ROS package containing the vehicle’s description plus the Deep-RL agents can be found in the Supplementary Material”  
但正文没有给GitHub链接 我也没有搜到明确稳定的公开仓库  
所以目前记录为：基础仿真框架开源 但作者完整HUAUV模型/RL代码是否公开只能说论文提到supplementary material 未确认有长期可访问仓库  

#### 观测输入
state总共26维：  
20维距离传感器采样  
3维previous actions  
3维target-related values  

距离传感器：  
空中使用LIDAR 模拟UST-10LX 10m range 270deg FOV 0.25deg resolution  
实际输入不是全分辨率scan 而是从270deg里每隔13.5deg采样 共20个range samples  

水下使用FLS sonar（Forward-Looking Sonar 前视声呐）  
模拟声呐20m range 1000 bins 256 beams beam width 90deg beam height 15deg  
policy同样只取20个beam 每个beam取最高bin 转成到障碍物或水箱墙的距离  

target-related values：  
机器人相对目标的位置  
x-y平面中相对目标的角度  
z-distance平面中相对目标的角度  

previous actions：  
把前三个动作也放进state 让policy知道近期控制趋势  
再加上LSTM/RNN结构 这说明作者认为这个任务有明显partial observability 只看当前range不够稳定  

需要注意：它虽然叫mapless 但不是“无定位”  
它不用显式全局坐标作为policy输入 但需要relative localization to target  
论文说真实里可以用GPS + USBL等标准定位传感器得到这类信息  

#### 输出动作
action是连续3维：  
linear velocity：0 到 0.25 m/s  
altitude velocity：-0.25 到 0.25 m/s  
delta yaw：-0.25 到 0.25 rad  
这里的altitude velocity可以理解成垂直方向速度命令 用来上下穿越水面或调整高度/深度 但它不是完整的heave/pitch/roll 6-DoF控制  

所以它比Digital Twin那篇的离散heading强一些  
但仍然不是底层thruster控制 也不是6-DoF wrench  
它是高层速度/航向命令 下面仍然依赖RotorS/UUV里的几何控制器或底层控制器执行  

#### Reward
reward非常简单 是binary sparse reward：  
到达目标距离阈值cd以内：+100  
碰撞、离障碍/墙小于co、或episode到500步超时：-10  
cd和co都设为0.5m  
如果没有到达也没有失败 中间step基本没有dense progress reward  

这说明policy主要靠大量仿真探索学导航  
没有显式奖励能耗、动作平滑、距离目标的连续进展、介质切换姿态稳定性等  

#### 实验结果
主要结论：double critic + RNN/LSTM比之前MLP single/double critic更能跨环境泛化  
环境1训练 环境2测试：  
A-W任务 stochastic SAC-RNN成功率94% deterministic TD3-RNN 73%  
之前Grando et al. 2021的MLP方法在环境2 A-W是0%  
BBA（Behavior-Based Algorithm 行为式算法）环境2 A-W是28%  

W-A任务明显更难：  
环境2中 TD3-RNN 8% SAC-RNN 10% BBA 8%  
如果环境1训练后再在环境2继续训练500 episodes SAC可以到83%  
所以它的“泛化”主要在A-W上比较强 W-A跨环境泛化其实很弱  

作者强调在真实仿真里有wind disturbance、hydrodynamics、thrusters和传感器模拟  
但所有结果都是仿真统计 不是实物实验  

#### limitation / future work
没有真实机器人实验  
作者最后说future studies with our real HUAUV are on the way  
所以它没有证明sim-to-real 只是说如果真实传感器和relative localization能保证 可能可用于真实系统  

任务依赖relative localization  
如果没有GPS/USBL/其他定位系统 机器人无法知道目标相对位置  
因此它不是纯靠声呐/LiDAR自主找目标  

水下感知仍然是模拟FLS sonar的稀疏range beam  
不是直接处理真实声呐图像/点云 也没有视觉  
这对复现友好 但感知难度被简化了  

动作是高层velocity/yaw command  
底层空中/水下控制、介质切换时的姿态稳定、推进器/旋翼切换等都由仿真模型和底层控制承担  
如果我们关心真实水下运动控制 这篇不能替代Learning to Swim/Fast Policy Learning那类低层控制工作  

reward很稀疏  
没有解释为什么策略在介质切换时一定稳定 也没有对失败模式做深入分析  

泛化范围有限  
所谓environmental generalization主要是从一个小水箱场景泛化到另一个小水箱/平台结构场景  
不是泛化到真实海洋、不同水流、不同声呐噪声、不同机器人动力学  

#### 对我们的启发
这篇比Digital Twin那篇更值得参考一点 因为它至少把navigation policy建在“range sensor + relative goal”的输入上 并且动作是连续速度/高度/yaw  
但它仍然是仿真高层导航 不是实物水下自主导航  

有价值的设计点：  
1）水下导航可以先用稀疏range而不是全图像/全声呐图 做低维RL输入  
2）RNN/LSTM对mapless navigation有意义 因为单帧range会丢失历史和运动趋势  
3）跨环境泛化必须单独测 不能只看训练环境成功率  
4）介质切换任务很难 尤其W-A比A-W难很多 可能因为从水下冲出水面时动力学和传感器切换更剧烈  

对我们自己的方向来说 如果做纯水下机器人 介质切换不是重点  
但可以借鉴“range/sonar + relative target + recurrent policy”的结构  
更进一步的gap是：把模拟range换成真实声呐/视觉输入 把外部relative localization换成机载估计 或者做teacher-student蒸馏  
链接：https://arxiv.org/abs/2209.06332

#### Deep Reinforcement Learning for Adaptive Path Planning and Control of an Autonomous Underwater Vehicle
2022 Hadi / Khosravi / Sarhadi  
发表/收录：Applied Ocean Research 129 (2022) 103326 DOI: 10.1016/j.apor.2022.103326  
这篇用TD3做REMUS-100 AUV的mapless motion planning + static obstacle avoidance 任务是从随机起点到随机目标 同时控制连续rudder angle避开障碍物  
它把规划决策和方向舵控制放进同一个policy 所以作者称end-to-end path planning and control  
但这里的“端到端”只是structured state -> rudder command 不包含camera/sonar原始感知 也不包含推进电机控制  

#### 任务和能力边界
任务在2D horizontal plane里进行：给定AUV当前状态、目标点和障碍物距离 输出方向舵角 让欠驱动REMUS-100以预设推进状态到达目标并避开静态圆形障碍物  
目标可以是single goal 也可以连续切换多个waypoints  
环境声称unknown / no prior knowledge 意思是没有预先给完整地图和固定路线 目标/障碍物在每个episode随机生成  
但它不是完整3D navigation：目标只有P_goal=[x_goal,y_goal] 深度由另一个独立agent保持常数  
它也不主动控制forward speed 作者把speed control列为future work  

#### 是否有动力学仿真
有 使用REMUS-100的nonlinear 6-DoF kinematic/dynamic equations：  
eta_dot = J(eta) nu  
M nu_dot + C(nu)nu + D(nu)nu + g(eta) = tau  
模型包含rigid-body inertia / added mass / Coriolis-centripetal / hydrodynamic damping / gravity-buoyancy / control surface effects  
参数来自作者之前的REMUS控制模型 工作在MATLAB里仿真 不是Isaac Lab/Gazebo这类场景仿真器  
海流通过relative velocity进入动力学 测试固定、突变和随机Gaussian current  

这里要准确区分：底层plant是6-DoF nonlinear dynamics 但规划任务仍是2D  
另一个depth agent用[e_z, integral(e_z), e_z_dot, w, q, theta]输出stern-plane/elevator command delta_s维持固定深度  
所以完整结构更像：2D TD3 planner/controller控制rudder + 独立depth controller + 预设forward propulsion  

#### 观测和“未知环境”的真实含义
主policy state维度是14+N：  
s_t = [AUV_state, d(AUV,Obs), P_goal, a_(t-1)]  
AUV_state=[u,v,w,p,q,r,phi,theta,psi,x,y] 共11维  
d(AUV,Obs)是AUV到N个已识别障碍物的Euclidean distance 共N维  
P_goal=[x_goal,y_goal] 共2维  
a_(t-1)是上一个rudder action 1维  

论文没有camera输入 也没有sonar image / point cloud / range scan作为network input  
它假设传感器范围内障碍物已经被识别 并直接把每个障碍物的精确距离交给policy  
所以不能说“所有障碍物在任务开始前全局已知” 更准确的说法是：地图事先未知 但局部感知结果被理想化为即时、准确的障碍物距离  
论文没有真正处理水下感知中的turbidity、低光、sonar speckle/multipath、遮挡、漏检/误检、量测噪声、data association和可变障碍物数量编码  
目标坐标和AUV的x/y状态也默认可以直接获得 没有研究无GPS条件下定位漂移如何影响导航  

这其实把真实系统中很难的一段直接抽象掉了：  
raw camera/sonar -> detection/ranging/localization -> structured obstacle distance/goal vector  
论文只从最后的structured vector开始训练navigation policy  

#### 输出动作
TD3 action是连续的：  
a = delta_r in [-25deg,25deg]  
delta_r是rudder angle / 方向舵命令 不是discrete left/straight/right 也不是thruster/motor command  
RL不控制推进器转速或目标speed 机器人以预设推进状态运动 nominal speed约1.5m/s maximum约2m/s  
实际u/v/w会受转弯、动力学和海流影响 因此不是数学上每时刻严格恒速 但policy没有速度控制通道  
另一个depth agent输出delta_s控制升降舵 但其目标只是constant depth  

#### 网络和训练
算法是TD3：continuous-action model-free off-policy actor-critic  
核心机制是twin critics取较小target Q、target policy smoothing和delayed actor update 用来缓解DDPG的Q overestimation和训练不稳定  
Actor和Critic结构相近 各有3个fully connected layers 每层256 neurons  
探索噪声使用Ornstein-Uhlenbeck process  
learning rate 0.001 discount factor 0.99 mini-batch 256  
每个episode最多500 steps 共训练10000 episodes  
硬件是AMD Ryzen 7 3800XT 8-core 3.89GHz 训练约60小时 约1500 episodes后开始学会到目标并避障 后期逐渐找到更短路径  
论文没有给代码/GitHub链接 当前未确认开源实现  

#### Reward
总reward：R_T = r1 + r2 + r3 + r4 + r5  
r1 goal reward：进入goal radius给r_goal 进入near-goal区域给较小正奖励 其他位置按distance to goal惩罚  
r2 obstacle reward：安全距离外给正奖励 进入d_O以内按侵入程度惩罚  
r3 heading error：惩罚当前yaw偏离start-to-goal initial LOS angle 促使AUV总体朝向目标  
r4 control effort：惩罚|delta_r| 减少大舵角  
r5 action fluctuation：惩罚当前action偏离过去tau个动作moving average 让rudder command更平滑  

reward同时考虑short path / collision avoidance / energy proxy / actuator saturation / smooth action  
但r4只是rudder magnitude regularization 不是根据真实motor current/power建模的能耗  
r3使用initial LOS而不是随当前位置更新的current LOS 在复杂绕障路径中可能和避障需求冲突  
Fig.11/12只比较是否加入r4/r5后的state/action曲线 不是与其他navigation algorithm的系统性benchmark  

#### 仿真环境和实验
训练场景是MATLAB里的2D rectangle  
REMUS长度1.4m 碰撞几何用radius 0.7m的circle近似  
每个障碍物用radius 1.5m circle表示 外加0.5m avoidance zone  
目标区域radius 3m 目标随机放在rectangle边界  
起点从中心5m x 5m区域随机采样 初始heading固定为0 障碍物和目标坐标随机  

测试包括：  
不同azimuth/距离的single target  
连续3个或5个waypoints  
随机static obstacles  
无海流  
初始0.3m/s 120deg 到第一个目标后突变为0.2m/s 30deg  
随机幅值和方向的ocean current  
20种movement scenarios对比travel time / path distance / minimum obstacle distance  
多数情况下海流增加时间和路程 少数顺流场景反而更快  

#### 是否有真实机器人
没有 全部结果是MATLAB simulation  
没有水池实验、海试、hardware-in-the-loop、真实range sensor输入或Sim2Real  
论文里的experiments都是simulation tests 结论中的realistic simulation不能理解成real-world validation  

#### Related works里不同“导航”工作实际做了什么
这篇related works把path planning / path following / collision avoidance / docking / position tracking都放在AUV navigation大类里 但它们解决的层级并不相同：  

下面的“实物”专指核心RL policy是否在真实载具闭环运行 不能把真实海流数据、电子海图或真实地形数据驱动的simulation误写成实物实验 也不能把只跟踪离线规划路径的field test写成RL obstacle avoidance的实物验证  

##### 1. Yoo and Kim 2016 — Path Optimization for Marine Vehicles in Ocean Currents Using Reinforcement Learning
任务：Q-learning为欠驱动marine vehicle寻找受海流影响的start-to-goal path 并与A*/D*比较  
实物：没有 全部是simulation；文中actual ocean current conditions指把真实海流场数据放入仿真 不是让真实船在海上运行  
主要简化：只用3-DoF kinematic/nonholonomic model 没有完整动力学和actuator dynamics；环境map和current-field data预先提供；没有obstacles 因而没有真实避障或感知问题；输出是规划层path/action 不是thruster/rudder的真实闭环控制  

##### 2. Blekas and Vlachos 2018 — RL-Based Path Planning for an Over-Actuated Floating Vehicle under Disturbances
任务：上层online LSPI选择desired velocity direction 下层PD velocity controller和control allocation驱动三角形over-actuated floating platform到目标  
实物：没有 明确只在simulated environment评估  
主要简化：对象是水面floating platform而非AUV；采用2D/3-DoF planar navigation；RL不直接输出thruster command而只给速度方向 底层跟踪由传统控制器解决；障碍物/目标和车辆状态是干净的结构化量 没有camera/sonar perception pipeline。相对其他早期工作它并非最粗糙：仿真加入platform dynamics、actuator limits/delay、measurement noise以及wind/wave/current 但这些仍是建模扰动 不是现场不确定性  

##### 3. Bhopale et al. 2019 — Reinforcement Learning Based Obstacle Avoidance for Autonomous Underwater Vehicle
任务：modified Q-learning控制AUV绕过single/multiple static obstacles 并用NN近似连续state下的Q function  
实物：没有 论文明确只做MATLAB simulations  
主要简化：action space仍是离散的；障碍物被简化成具有已知安全边界的几何体；人工定义unsafe zone 进入危险区后直接停止exploration、只执行exploitation 相当于额外注入安全规则；只验证静态障碍；没有原始sonar/camera、sensor noise、localization drift和动态目标  

##### 4. Sun et al. 2020 — AUV 3D Path Planning Based on the Improved Hierarchical Deep Q Network
任务：hierarchical DQN + prioritized replay + artificial potential field在离散化3D地形中规划路径 把任务分层以缩小state/action search space  
实物：有field experiment 但只能算“规划路径跟踪的部分实物验证” 不能算learned obstacle avoidance的实物验证。作者把仿真得到的path nodes发送给真实AUV；现场避障另用manual experience rule：距离小于3 m时按经验改变航向；而记录中sonar距离一直大于3 m 该规则也从未触发  
主要简化：环境被离散成triangular-prism/grid nodes；水平动作是12个相隔30 deg的方向 再加离散vertical action；uniform current代替复杂时变流场；APF把目标方向等先验直接写入reward；实物定位依赖DVL/compass/depth/altimeter dead reckoning 并两次上浮用GPS校正；没有展示策略直接处理真实sonar回波、定位漂移和真实障碍遭遇  

##### 5. Zhang et al. 2020 — Deep Interactive Reinforcement Learning for Path Following of Autonomous Underwater Vehicle
任务：DQN学习straight-line和sinusoidal-curve path following 比较environment reward、human reward以及二者结合的interactive RL  
实物：没有 全部在Gazebo/UUV Simulator中完成 作者把actual AUV deployment列为future work  
主要简化：这是path following而不是unknown-environment navigation 没有obstacles；state只有cross-track distance、course以及曲线任务中的tangent slope/desired course；action是若干离散rudder-angle values；human trainer通过RViz观察模拟AUV后人工打分；没有camera/sonar input、state-estimation error、current disturbance或sim2real  

##### 6. Zhao and Roh 2019 — COLREGs-Compliant Multiship Collision Avoidance Based on Deep Reinforcement Learning
任务：policy-gradient DRL依据相遇船状态输出own-ship rudder angle 让多艘水面船遵守COLREGs并避碰  
实物：没有 只在多种simulated encounter scenarios中验证  
主要简化：研究对象是surface ships而非AUV；各船仍沿predefined paths朝各自目标航行 RL主要做局部collision-avoidance maneuver；周围船的相对状态被直接提供 类似AIS/radar级结构化输入 这种AIS信息水下不可用；为限制输入维度 把目标船分入四个COLREGs区域且每区只保留最近一艘；没有真实sensor association/occlusion/communication loss和船舶实测执行误差  

##### 7. Cheng and Zhang 2018 — Concise Deep Reinforcement Learning Obstacle Avoidance for Underactuated Unmanned Marine Vessels
任务：DQN/CNN为underactuated unmanned marine vessel学习target approaching、obstacle avoidance、speed modification和attitude correction  
实物：没有 文中的experiments是Theano环境中的illustrative numerical experiments  
主要简化：case study是autonomous surface vessel 不是水下AUV；仅考虑surge/sway/yaw的3-DoF horizontal dynamics；DQN只能输出少量离散且固定持续时间的control behaviors；sonar detection result和vehicle state经data-fusion后才交给网络 并未建立真实sonar成像、噪声、遮挡和误检模型；环境和动力学参数固定 所谓unknown dynamics指policy设计时不显式使用模型 不代表训练环境没有模型  

##### 8. Anderlini et al. 2019 — Docking Control of an Autonomous Underwater Vehicle Using Reinforcement Learning
任务：用continuous-action DDPG和discrete-action DQN控制REMUS-100靠近固定docking platform 并与PID和optimal control比较  
实物：没有 作者明确说明是在simulation environment中研究  
主要简化：把6-DoF问题限制在vertical plane 只保留surge/heave/pitch 3 DoF；忽略ocean current；docking platform固定；直接提供相对position/velocity等状态 等价于假设视觉或声学定位已经可靠完成；没有研究目标检测、声学/视觉噪声、moving dock、机械接触/捕获和真实执行器；这是近距离terminal control 不是一般未知环境navigation  

##### 9. Carlucho et al. 2018 — AUV Position Tracking Control Using End-to-End Deep Reinforcement Learning
任务：goal-conditioned DDPG把低维sensor/state information直接映射到Nessie VII的6个thruster commands 让AUV到达可变化的3D waypoint  
实物：没有 使用真实存在的Nessie VII作为建模对象 但所有结果来自其dynamic-model simulation；论文把real-robot deployment列为future work  
主要简化：虽然控制6个thrusters 载具本身只有5-DoF actuation；目标是无障碍的point reaching 不是collision-free path planning；policy获得AUV状态和目标关系 不处理raw visual/sonar perception；没有海流和定位误差验证；robustness只通过simulated T1 thruster 90% thrust loss展示；训练起点固定 目标区域用curriculum从radius 1.5 m缩到0.5 m  

##### 10. Sun et al. 2019 — Mapless Motion Planning System for an AUV Using Policy-Gradient DRL
任务：policy-gradient/PPO-style continuous policy以sensor/goal state为输入 输出continuous surge force和yaw moment 连续到达多个目标并避障 还加入reward curriculum和current test  
实物：没有 论文结论只报告simulation results  
主要简化：只做固定深度的3-DoF horizontal motion；输出只有surge和yaw 其余自由度不进入navigation task；传感器信息是已处理的低维障碍距离/目标关系 而非raw sonar；障碍物是仿真几何体；定位、检测、data association和通信问题不在研究范围；所谓mapless只是没有预先构建global map 不等于解决真实水下感知  

##### 11. Havenstrom et al. 2021 — DRL Controller for 3D Path Following and Collision Avoidance by AUVs
任务：先用传统waypoint interpolation/QPMI生成smooth 3D reference path 再由PPO同时做path following和reactive collision avoidance  
实物：没有 全部是simulation results  
主要简化：AUV使用6-DoF nonlinear dynamics 但policy只输出propeller shaft speed、rudder和elevator三个actuator signals；forward-looking sonar被理想化成15 x 15 exact distance image 没有真实声呐散斑、多径、盲区、漏检和时延；障碍物是合成几何体；ocean current方向在每个episode内固定且irrotational 风浪被忽略；global path假设已经由别的planner生成 所以RL不是完整global planner；甚至dead-end场景中偏重path-following的policy会失败  

##### 12. Guo et al. 2020 — An Autonomous Path Planning Model for Unmanned Ships Based on DRL
任务：DDPG结合virtual/artificial potential field 在电子海图中输出heading deflection和speed increment 完成水面多船COLREGs collision avoidance  
实物：没有 AIS数据和电子海图是真实格式的数据源 但训练和验证都是Python + electronic-chart simulation  
主要简化：研究对象是surface ship而非AUV；用二维800 x 600 pixel chart和理想化heading/speed update 没有hydrodynamic/actuator model；state直接包含自身、目标和周围船信息 没有range finder或raw perception；1000 m内只保留最危险船的距离 会丢失多目标交互信息；COLREGs被转化成手工navigation restricted area APF又把专家先验写入reward；AIS可在水面使用 但不能直接迁移到水下  

##### 13. Sun et al. 2021 — A 2D Optimal Path Planning Algorithm for AUV Driving in Unknown Underwater Canyons
任务：DDPG/SumTree-DDPG根据7束sonar distance输出continuous forward speed和yaw-related command 在二维峡谷中避开静态canyon walls与模拟动态障碍  
实物：没有 作者用Python自建underwater-canyon simulation并只报告data simulation  
主要简化：作者明确把任务限制为horizontal 3-DoF和0.5 s离散控制周期；7个sonar直接返回0–150 m精确距离 没有声学成像与噪声/遮挡；所谓underwater canyon是人工生成的2D边界；不考虑3D地形机动、energy、waves和ocean current；运动/action formulation没有严格满足真实underactuated AUV的nonholonomic/dynamic constraints 因而路径可行性与实艇可执行性之间仍有缺口  

##### 实物验证核对结论
上述13篇中 12篇是纯仿真  
Sun et al. 2020是唯一包含真实AUV field experiment的工作 但实物环节只是跟踪由仿真规划得到的path nodes 核心HDQN没有使用真实sonar闭环避障 现场的rule-based avoidance也没有被触发  
因此严格来说 这13篇没有任何一篇证明了“RL policy读取真实水下障碍感知并闭环控制真实AUV成功避障”  

核对入口：  
[Yoo and Kim 2016](https://doi.org/10.1007/s00773-015-0355-9) / [Blekas and Vlachos 2018](https://doi.org/10.1016/j.robot.2017.12.009) / [Bhopale et al. 2019](https://doi.org/10.1007/s11804-019-00089-3) / [Sun et al. 2020](https://doi.org/10.3390/jmse8020145) / [Zhang et al. 2020](https://doi.org/10.1109/ACCESS.2020.2970433) / [Zhao and Roh 2019](https://doi.org/10.1016/j.oceaneng.2019.106436) / [Cheng and Zhang 2018](https://doi.org/10.1016/j.neucom.2017.06.066)  
[Anderlini et al. 2019](https://doi.org/10.3390/app9173456) / [Carlucho et al. 2018](https://doi.org/10.1109/OCEANS.2018.8604791) / [Sun et al. 2019](https://doi.org/10.1007/s10846-019-01004-2) / [Havenstrom et al. 2021](https://doi.org/10.3389/frobt.2021.739013) / [Guo et al. 2020](https://doi.org/10.3390/s20020426) / [Sun et al. 2021](https://doi.org/10.3390/jmse9030252)  

#### 对“这些都是导航论文”的批判性判断
你的感觉基本是对的：这批早期工作中的核心RL policy全部缺少真实水下闭环避障验证 而且把真实水下导航问题大幅简化  
但不宜全部叫“纸上谈兵” 它们有价值的地方是分别研究continuous RL、reward shaping、hierarchy、current robustness或dynamic constraints等单个模块  
问题在于论文标题常把这些局部模块统称navigation 容易让人误以为已经完成完整自主系统  

真正的水下闭环navigation至少包含：  
1）perception：camera/sonar/DVL/depth/IMU获取有噪声观测  
2）state estimation：在无GPS条件下估计自身pose/velocity和goal relation  
3）environment representation：从观测得到obstacle geometry/free space 处理occlusion和动态目标  
4）planning/decision：生成collision-free且dynamically feasible的local/global motion command  
5）low-level control：把velocity/pose/wrench command稳定转成rudder/thruster output  
6）real-world robustness：处理current、turbidity、actuator delay/failure、localization drift和通信限制  

多数related works只解决第4层 或第4+第5层 并直接假设第1-3层已经给出干净的低维state  
因此“mapless”通常只表示没有预先存储global map 不代表机器人真正从raw sensor解决了未知环境感知  
“6-DoF model”也不代表任务是3D navigation；本篇就是6-DoF plant + 2D planner + constant-depth agent  
“end-to-end”也不代表image-to-motor；本篇只是structured distances/state-to-rudder  

如果按真实程度分层 可以这样看：  
Level 0 privileged-state planning：直接给精确位置/目标/障碍物几何 在简单kinematic simulation里规划  
Level 1 idealized sensor navigation：给range/distance vector 但没有sensor physics/noise/occlusion 本篇大致在这里  
Level 2 dynamics-aware simulation：加入6-DoF hydrodynamics/current/actuator limits 本篇底层模型也达到这一层  
Level 3 real pool closed loop：真实DVL/IMU/camera/sonar反馈 水池里完成感知-决策-控制  
Level 4 open-water autonomy：在真实海洋长期运行 处理能见度、海流、定位漂移和任务变化 Nav2Goal的真实海试比多数上述工作更接近这一层  

#### limitation / future work
静态圆形障碍物 + 精确距离输入 距真实camera/sonar感知差距很大  
神经网络输入写成14+N 但实际MLP需要固定输入维度 论文没有充分解释障碍物数量变化时的padding/order/set encoding  
规划只在2D水平面 深度固定 动态障碍物和完整3D避障被列为future work  
RL只输出rudder 不控制speed 遇到近障碍时不能主动减速 作者也把speed control列为future work  
没有真实定位误差、range noise、sensor latency、motor/rudder dynamics误差和通信问题  
海流模型相对简单 而且没有与经典planner/controller或其他DRL做充分定量比较  
训练约60小时 效率远低于后来的Isaac/MJX massively parallel方法  
没有实物验证和Sim2Real 也没有确认代码开源  

#### 对我们的启发
这篇最值得借鉴的是把goal progress、obstacle clearance、heading、action magnitude和action smoothness放进同一个reward 并使用continuous rudder action  
如果我们后续做真正水下navigation 不应该继续把perfect obstacle distance当作最终输入 而可以把它作为teacher/privileged policy的输入：  
teacher：perfect pose + obstacle geometry -> safe velocity/rudder command  
student：raw/processed sonar + camera + DVL/IMU history -> imitate teacher or继续RL  
同时使用RNN/transformer/history处理partial observability 用realistic sonar/camera noise和localization drift做domain randomization  
系统结构上最好把navigation和control分层：sensor policy输出local waypoint/velocity Sim2Swim或其他low-level controller负责稳定执行 这样更容易从仿真迁移到真实机器人  
链接：https://doi.org/10.1016/j.apor.2022.103326

#### Reinforcement Learning for Autonomous Underwater Vehicles via Data-Informed Domain Randomization
2023 Harbin Institute of Technology Shenzhen / University of Hong Kong  
发表/收录：Applied Sciences 13(3):1723 DOI: 10.3390/app13031723  
这篇研究的不是navigation / obstacle avoidance 而是把Webots里训练的AUV定点控制器迁移到动力学不同的真实水下机器人  
核心方法叫RL-DDR：先用SAC在仿真中训练source controller 再利用少量真实机器人轨迹学习仿真控制量与真实控制量之间的映射 让控制器适应payload变化、thruster degradation、channel mismatch、dead zone和saturation等动力学变化  

##### 任务和能力边界
论文把任务称为pose regulation 目标是让机器人从随机初始状态回到earth-fixed frame的原点并停稳  
机器人模型只保留4个可控DoF：surge / sway / heave三个平移方向和yaw旋转  
roll和pitch不由policy控制 而是假设通过CoB-CoM配置产生足够的restoring force 被动保持接近0  

更严格地看 reward里的regulation error e是3维位置误差 没有单独给定任意desired yaw reference  
因此它更接近“用4个控制通道完成3D position regulation并抑制yaw rate” 而不是任意4-DoF pose tracking 更不是完整6-DoF控制  

它不做路径规划、waypoint navigation、obstacle avoidance或trajectory tracking 也没有camera/sonar perception输入  
所以这篇应放在low-level control / Sim2Real adaptation线 而不是learning-based navigation线  

##### DDR是什么意思
DDR是Data-informed Domain Randomization 可以译为“数据引导的域随机化”或“由真实数据提供信息的域随机化”  

普通Domain Randomization在训练前人为给仿真参数设置随机范围：  
mass / added mass  
drag和Coriolis参数  
gravity / buoyancy  
thruster-to-wrench mapping  
thruster dead zone和saturation  
policy在大量随机动力学中训练 从而期望真实机器人动力学落在训练分布里面  

问题是实际机器人可能出现仿真参数集合根本没有覆盖的变化 比如推进器接线反向、某个推进器损坏、海草缠绕导致推力衰减或者未建模的thruster dynamics  
即使把参数随机范围设得很大 也不能保证真实target dynamics属于source simulator能够表达的模型族  

这篇DDR不只是在真实数据基础上重新调整随机参数分布 其实际关键是学习一个小型control/action mapping：  
simulated source action <-> real target action  
也就是寻找“仿真器里什么控制量”和“真实机器人上什么PWM”会产生近似相同的state transition  
可以把它理解成仿真policy和真实推进器之间的控制指令翻译器  

作者使用仿真的forward / inverse dynamics网络以及少量target trajectory训练映射G/H/R  
当映射误差较小时持续gradient update 当误差突然变大时清空旧数据并重新收集一小批样本 从头训练这个小网络  
因为网络很小 作者称重新训练通常少于1秒  

因此这里的DDR本质更接近：  
domain randomization + data-driven action alignment + online dynamics adaptation  
论文中的G/H/R/F符号方向和训练顺序有一些前后不一致 但核心思想就是不重新训练整个SAC policy 而只快速更新source/target之间的动作适配层  

##### Observation / action / reward
policy输入是当前位置相对原点的body-frame regulation error以及4维generalized velocity  
论文假设这些状态能够以较高频率、较小噪声直接获得 不涉及水下定位算法本身  

SAC policy输出连续4维控制量  
理论表达为generalized wrench：tau=[tau_x,tau_y,tau_z,T_z] 对应三个方向的force和yaw torque  
Webots和BlueROV2实现中写成4维PWM control vector 再映射到六个推进器  
所以它不是离散动作 也不是直接独立输出六个motor RPM 更接近输出四个运动通道的连续PWM/wrench后再做thruster allocation  

reward很简单：  
r = -alpha_1||e|| - alpha_2||nu|| - alpha_3||tau||  
alpha_1 >> alpha_2 >> alpha_3 > 0  
分别要求机器人靠近原点、到达后停止运动、避免使用过大的控制量  
没有obstacle、path length或sensor-related reward  

##### 仿真器和训练
source domain使用Webots 模拟AUV质量12kg neutral buoyancy 配置六个推进器  
每个推进器最大推力约[-50,50]N dead band设为[-5,5]N  
动力学包括mass / added mass、linear and quadratic drag、gravity / buoyancy以及输入dead zone / saturation 但实验模型把Coriolis matrix设为0 并没有模拟复杂thruster dynamics  

source controller使用continuous-action off-policy Soft Actor-Critic训练  
训练episode的初始xyz位置从[-10,10]m立方体随机采样 初始线速度从[-2,2]m/s范围随机采样  
论文给出了nominal dynamics参数 但没有清晰报告完整domain-randomization distribution和所有随机范围 复现性有限  

##### Sim-to-Sim动力学失配实验
作者人为构造三种target dynamics：  
1）交换两个推进器的control channel / installation mapping  
2）降低若干推进器的gain和maximum thrust 模拟海草缠绕或性能退化  
3）一个水平推进器损坏 另一个推进器PWM相位反转 导致实际推力方向相反  

未经适配的source policy在target dynamics中出现较大误差甚至无法稳定  
DDR用几个episode的数据学到action mapping后重新实现定点稳定 每个数据收集episode约20秒  
100组初始状态测试中 transferred policy的mean error约为0.019m / 0.028m / 0.021m 对应未适配policy约0.41m / 0.39m / 1.16m  
但如果两个水平推进器同时损坏 系统剩余可控性不足 方法也会失败  

需要注意 上述推进器交换、退化、损坏和反转都只在sim-to-sim中验证 没有在真实机器人上进行对应故障实验  

##### Sim-to-Real实物实验
真实平台是六推进器BlueROV2 但严格说是tethered ROV而不是完全自主AUV  
机器人通过ROS和岸上laptop通信 laptop运行policy并发送PWM  
状态反馈来自水池墙上12台水下相机组成的Nokov optical motion-capture system 以60Hz提供全局position和quaternion 反光marker直径30mm  
所以论文绕开了真实水下自主定位、DVL/IMU融合、声呐定位、浑浊水体和开放水域通信问题  
作者也明确说未来需要加入sonar-based localization和更新硬件 才能更接近真正AUV  

水池里用固定propeller制造未知外部扰动 真实机器人从[0.18,-0.46,-0.40]m附近回到原点 初始距离约0.64m  
未适配source controller不能稳定在原点 transferred controller能够完成定点稳定  
正文声称最终稳定在radius 0.15mm的球内 但这个数值比其sub-centimeter motion-capture resolution还小 且对水中BlueROV2明显不合理 很可能是0.15m的单位/排版错误 不能直接作为可信精度引用  

##### 是否zero-shot
不是严格zero-shot sim-to-real  
部署过程必须在target robot上采集少量state transition / control数据 再训练action mapping得到target controller  
更准确的分类是few-shot sim-to-real / online controller adaptation  

和后续工作的区别：  
Learning to Swim / Sim2Swim通过massively parallel training和domain randomization直接zero-shot部署  
CORAL-AUV通过CFD surrogate drag model提高训练动力学真实性再zero-shot部署  
RL-DDR则接受source simulator与target robot存在失配 用少量真实数据在线修正动作映射  

##### limitations
真实验证是短距离受控水池定点实验 不是开放水域、长距离trajectory或navigation  
只控制4个运动通道 roll/pitch依赖被动静稳性 没有任意姿态和完整6-DoF控制  
真实系统是tethered BlueROV2 + external laptop + 12-camera motion capture 不能等价成自主AUV海试  
真实实验没有验证payload突变、thruster failure或反转 这些关键鲁棒性结果主要来自仿真  
在线适配需要让尚未适配的controller在target robot上运行并收集数据 这一阶段可能不稳定或损伤硬件 论文没有safety filter / backup controller  
没有与PID、robust/adaptive control、普通DR和其他transfer RL方法进行充分同平台实物benchmark  
真实实验主要展示单条trajectory和error curve 缺少多次试验统计、success rate和不同扰动强度评估  
方法假设source和target具有相同state/action维度以及相同任务 objective 无法直接处理传感器或机器人结构改变  

##### 对我们的启发
这篇最值得保留的是在通用RL controller与真实硬件之间加入快速在线适配层  
如果未来做水下navigation 可以让高层planner输出desired velocity / pose 底层用Learning-to-Swim或Sim2Swim式policy 再在policy与thruster allocation之间增加DDR-style action adapter 处理载荷、推进器老化和实际水动力变化  
也可以把CORAL-AUV的CFD surrogate作为高保真prior 再用DDR-style少量在线数据校正剩余sim-to-real gap  
但实际部署必须加入safe exploration / fallback controller 并用onboard DVL / IMU / pressure / sonar状态估计替代外部motion capture 才能说明方法适用于真实AUV  

链接：https://doi.org/10.3390/app13031723  

#### Dynamic Robotic Tracking of Underwater Targets Using Reinforcement Learning
2023 ICM-CSIC / MBARI / UPC-BarcelonaTech / Barcelona Supercomputing Center / University of Girona  
发表/收录：Science Robotics 8(80), eade7811；2023-07-26  
DOI：https://doi.org/10.1126/scirobotics.ade7811  

这篇是high-level guidance / informative path planning论文 不是低层AUV control  
系统让水面ASV利用acoustic modem对一个水下移动目标进行range-only tracking  
RL不直接预测目标位置 也不控制水下target 而是决定水面tracker下一步朝什么方向运动/去哪个waypoint 从新的测量位置再次进行声学测距  
核心问题可以概括为：为了让水下目标的位置估计更准确 水面机器人下一次应该去哪里测量  

##### 任务和两个机器人
tracker / agent是水面ASV 真实实验使用Liquid Robotics Wave Glider  
target是水下移动目标 可以是携带acoustic tag的动物或AUV 真实实验使用MBARI LRAUV  

Wave Glider在水面能够利用GPS获得自身位置 LRAUV在水下不能连续使用GPS 但两台机器人都装有acoustic modem  
modem通过信号往返传播时间给出两者之间的距离 但不直接给出目标方位、二维坐标或完整速度  
论文还假设target depth已知 因而可以把三维斜距投影为horizontal range  

一次range measurement只能说明目标位于一个圆周上  
只有tracker知道自己在多次测量时的位置 并从不同位置取得多个range 才能通过LS/PF triangulation估计target position  
因此GPS的作用不仅是让ASV执行waypoint 还为每个range measurement提供已知测量基线 并把target trajectory放入全球地理坐标  
理论上GPS可以被准确relative odometry替代 但必须知道tracker在连续测量之间的位移与航向  

##### 系统pipeline和任务层级
系统同时运行两个彼此独立的算法：  
1）target state estimation：range measurements + tracker GPS position + known target depth -> LS/PF target position estimate  
2）RL path planning：current observation + estimated target relative position -> next heading / waypoint  

完整结构是：  
ASV GPS + acoustic range -> LS/PF estimator -> estimated target position -> H-LSTM-SAC guidance -> next waypoint -> Wave Glider原有autopilot -> ASV实际移动 -> 新测距  

论文明确把autonomous navigation分为guidance / navigation / control三层  
RL只位于guidance层负责决定path/waypoint  
平台自己的navigation和control层负责定位、维持速度并跟踪waypoint 因此policy不绑定Wave Glider的具体推进和转向机构  

它不同于Hadi 2022把planning和rudder control放入同一TD3 policy的做法  
这篇选择更高层的抽象动作 所以更容易从particle simulator迁移到真实平台 但也默认已有的low-level waypoint controller基本可靠  

##### Agent / target / measurement model
训练环境先使用简化的2D particle kinematics 而不是完整ASV/AUV hydrodynamics  
无海流时agent position满足p_dot=v velocity变化由force/mass决定  
实际任务把agent speed固定为1m/s action只改变yaw  
每个decision interval为30s 新yaw和固定速度共同确定下一个waypoint  
可以理解为piecewise-constant heading：每30s重新选择一次方向 中间由底层controller执行  

target训练时的速度和方向在episode开始随机初始化 单个episode内基本保持constant velocity  
测试还使用Lévy-flight random walk产生训练中没有见过的突变运动 作为generalization evaluation  

声学测距模型是：  
d_t = ||p_t - q_t|| + w_t  
w_t服从带systematic bias的Gaussian noise 训练设置sigma约1m bias约为真实距离的1%  
如果已知target depth d_q 则horizontal projected range可由sqrt(d_t^2-d_q^2)获得  

仿真还加入：  
10% acoustic measurement / communication dropout  
超过最大通信距离后完全没有range measurement  
随机方向和速度的ocean current 最高达到agent speed的一半  
但海流只是通过修改particle position近似 没有完整wave/current/vehicle dynamics  

##### Target position estimator
训练阶段使用unconstrained Least Squares  
LS通过对多个squared range equations线性化来估计unknown target position 计算速度比PF快几个数量级 适合百万级RL rollout  
LS适合static / slow target 但target速度增加后精度迅速恶化  

测试和真实部署同时考虑Particle Filter  
PF利用target motion model和历史measurements维护可能位置的particle distribution 对moving target明显优于LS 但计算成本更高  

RL与estimator不是end-to-end训练  
network输入的是LS/PF已经给出的target estimate / relative vector 而不是raw acoustic waveform、sonar image或modem packet  

##### Observation和action
每个time step的observation为：  
o_t = [p_t, v_t, d_hat_t, d_projected_t, d_depth, p_o]  
p_t：ASV当前2D position 真实部署来自GPS  
v_t：ASV velocity  
d_hat_t = p_t - q_hat_t：ASV相对estimated target position的vector  
d_projected_t：当前声学斜距投影到水平面的range  
d_depth：known target depth  
p_o：最后一次成功获得range measurement时ASV所在的位置  

p_o用于communication dropout：连续测距失败时 policy仍知道最后一次可靠观测发生在哪里  
对H-LSTM版本 hidden state还保存更长时间的历史信息  

action是连续一维yaw control / heading variation a_t=u_psi  
policy不控制forward speed、不输出propeller或rudder command、不直接输出target coordinate  
action与固定速度和30s time step结合生成下一waypoint 然后由ASV原有control stack执行  

##### Reward
总reward由三部分组成：  
r = r_d + r_e + r_terminal  

r_d根据ASV与estimated target之间的距离给奖励 促使tracker快速接近并保持足够好的acoustic link  
r_e根据target estimation error ||q_hat_t-q_t||给奖励 促使policy选择能够改善triangulation geometry和observability的测量位置  
r_terminal在距离超过d_max时给较大惩罚 避免失去声学链路；在距离低于d_min时惩罚 避免tracker与target过近/碰撞  

这里q_t的真实target position只在simulation training中用来计算privileged reward  
真实部署不需要继续计算reward policy只使用estimator给出的q_hat_t  
因此不能说网络从真实海试中的target ground truth进行online learning 真实部署没有finetune  

##### RL算法和memory
作者实现并比较DDPG、SAC、LSTM-DDPG、LSTM-SAC和H-LSTM-SAC  
DDPG与SAC都是continuous-action off-policy actor-critic 可以通过replay buffer重复利用历史transition  
SAC在最大化expected reward的同时最大化policy entropy 相比DDPG训练更稳定、探索更多  

普通LSTM-SAC显式输入过去l个observation sequence  
H-LSTM-SAC在SAC隐藏层中放入single-cell LSTM 并把internal hidden state传给下一time step history length设为1但记忆由hidden state持续传播  
range-only tracking是POMDP 单个时刻的一次距离无法确定target direction和velocity 所以memory具有合理性  

训练初期policy几乎走直线 target estimate很差  
随后先学会接近target 但collision增加  
再逐渐学会围绕estimated target绕圈 从不同方位取得range measurements 以改善triangulation geometry  
约0.6 million episodes时已经会在target附近绕圈但圆心存在偏差 后续才学会以estimated target为中心环绕  
普通SAC约1 million后停止改善 H-LSTM-SAC继续改善到约2 million并学到更好的collision avoidance；普通LSTM-SAC后期性能反而下降  

##### Simulation benchmark
agent speed设为1m/s target speed训练范围为agent speed的0到1/3  
每个episode 200 steps 每步30s 对应约1.67h真实任务时间  
每种测试重复100次并报告median target error、distance以及相对predefined path的probability of improvement  

解析baseline是在target上方以固定半径均匀采样的circle path 其半径至少为sqrt(2) times target depth 实验使用180m  
static target + no current的理想条件下 analytic circle最好 steady-state error约0.2m SAC/H-LSTM约0.4到0.6m 说明RL没有超过已知条件下的解析最优解  
加入current或moving target后 fixed circle不能及时适应 SAC/H-LSTM能够更快接近并维持distance  
高速target时LS误差明显增加 PF更适合moving target  
训练没见过的Lévy-flight random walk测试仍能跟踪 表明对target motion有一定zero-shot generalization 但不代表对任意现实环境的理论保证  

作者还在Iqua Robotics提供的high-fidelity environment中测试 realistic commercial vehicle dynamics、communication和command/control architecture 作为particle simulation与真实海试之间的中间验证  

##### 真实海试
真实实验在California Monterey Bay 距海岸约5到10km的开放海域进行  
tracker是搭载MBARI Hotspot payload的Wave Glider target是搭载imaging payload的MBARI LRAUV  
Hotspot负责在underwater acoustic communication和cellular/satellite communication之间进行relay  

LRAUV在水下30到50m执行drift mission和keep-station mission  
drift mission让LRAUV随current漂移  
keep-station mission中如果LRAUV漂离指定station过远 就高速直线返回 导致velocity和heading突然变化 是range-only PF更困难的情况  

LRAUV约以60min communication threshold周期性上浮 建立radio communication并取得GPS fix  
上浮主要用于自身导航校正、任务通信和安全 不代替两次上浮之间的continuous acoustic tracking 论文主文没有进一步说明精确的watchdog触发逻辑  

一次实验中Wave Glider跟踪约2.5km / 5h  
另一类keep-station任务中PF曾因LRAUV突然高速返航而短暂丢失目标 但Wave Glider仍维持约9.5h任务跟踪  
全部field deployments累计超过15h和4km  
policy从simulation直接用于真实Wave Glider 没有针对具体field setup fine-tuning  

##### 是否考虑ASV obstacle avoidance
没有  
论文reward中的collision主要指tracker与target之间距离低于d_min 不是避开船舶、浮标、礁石、海岸或海上设施  
observation没有AIS、radar、camera、sonar obstacle range或电子海图输入  
因此RL policy无法感知并主动遵守COLREGs或避开环境障碍 真实任务主要选择开放海域并依赖平台原有安全操作与人工监督 但论文没有把这些设计成算法模块  

要形成完整海上系统 应在RL tracking waypoint与low-level control之间加入独立safety / collision-avoidance layer：AIS/radar/chart/vision -> COLREGs and obstacle check -> safe modified waypoint  

##### 真实能力边界和limitations
跟踪者是water-surface ASV 能持续使用GPS 并不是一个自身也GPS-denied的underwater tracker  
问题被限制为2D target localization 且target depth已知  
target必须携带cooperative acoustic modem/transponder 不能直接跟踪完全无标签、无回复的任意动物或AUV  
它不是imaging-sonar tracking：没有从回波中进行detection / classification / data association  
agent保持constant speed policy只控制heading 没有联合速度/能耗优化  
训练的particle model非常简化 current通过position offset近似 没有真实Wave Glider hydrodynamics、wave propulsion和actuator constraints  
RL依赖现有waypoint controller能够基本执行命令 低层tracking error由下一时刻实际GPS反馈再纠正 但严重control failure不在方法范围内  
没有environmental obstacle avoidance和COLREGs  
只验证single tracker + single target 没有multi-ASV cooperative ranging  
高速moving target时估计误差仍可能达到几十到上百米 属于远距离acoustic tracking 不是近距离精密跟随  

##### 开源
作者开源了训练算法和particle environment：  
https://doi.org/10.5281/zenodo.8063918  
也开源了部署RL方法的ROS implementation：  
https://doi.org/10.5281/zenodo.8063968  

##### 对我们的启发
这是比许多只做MATLAB/Gazebo仿真的水下navigation论文更可信的field robotics工作 关键不是更复杂的network 而是选择了合适的task abstraction  
RL负责具有不确定性和信息价值权衡的high-level measurement-location decision 传统LS/PF负责target estimation 成熟autopilot负责low-level execution 这种模块化结构比raw sensor -> thruster的端到端路线更容易实海部署  

对后续水下navigation很有价值的思想是active perception：机器人移动不只是为了接近goal 也可以为了让后续measurement更有信息  
如果扩展到真正underwater tracker 需要联合处理tracker自身DVL/INS drift、target uncertainty和acoustic ranging 可能变成joint localization and tracking / belief-space planning  
如果扩展到真实海上自主系统 还需要加入AIS/radar/sonar/chart-based obstacle avoidance与COLREGs safety layer  

#### Adaptive Formation Motion Planning and Control of Autonomous Underwater Vehicles Using Deep Reinforcement Learning
Hadi / Khosravi / Sarhadi  
发表/收录：IEEE Journal of Oceanic Engineering Vol.49 No.1 pp.311–328；online publication 2023-08-09，正式卷期为January 2024  
DOI：https://doi.org/10.1109/JOE.2023.3278290  

这篇可以理解为作者2022 single-AUV TD3 navigation工作的multi-AUV extension：  
2022工作让一艘REMUS-100到达二维目标并避障 这篇让一个leader和两个followers组成三角编队共同到达目标  
leader负责goal reaching和引导整体运动 followers负责维持相对距离与角度 同时系统还要避免environmental obstacles和AUV之间的self-collision  
论文把formation planning、formation control、obstacle avoidance和low-level actuation放进同一套TD3框架 这是它相对2022 single-AUV工作的主要扩展  

#### 任务到底是什么
实验中有3艘underactuated REMUS-type AUV：1个leader + 2个followers  
三艘艇在constant-depth horizontal plane中运动 最终形成并保持triangular formation  
leader的任务是朝随机目标区域运动并避开障碍物  
followers的任务是跟随leader 将leader-follower distance保持在25 m 将formation angle保持在150 deg 同时避开障碍物和其他AUV  
目标是半径3 m的圆形区域 放置在500 m x 500 m训练区边界  

所以它并不是传统意义上的global multi-robot path planner：  
policy没有输出整条trajectory或一串waypoints 而是在每个time step直接输出propeller thrust和rudder angle  
更准确地说 它是multi-agent reactive motion planning + formation control + low-level actuation的联合策略  

#### 两种避障架构
Approach 1：每艘AUV各自避障  
leader和两个followers都假设装有forward-looking sonar 每艘艇都拥有自己的TD3 policy、obstacle-distance input和obstacle-avoidance reward  
遇到障碍物时collision avoidance优先于formation keeping follower可以暂时脱离编队 绕开障碍后再恢复desired distance/angle  
通信主要是leader把自身实时状态单向发送给followers leader不需要followers状态  
这种方法更接近distributed local avoidance 传感器和计算需求较高 但单艇面对局部障碍时更安全  

Approach 2：只有leader负责整体避障  
只有leader具有obstacle-detection module followers只做formation tracking  
leader根据三艘艇的位置计算formation triangle的circumscribed circle 用整个外接圆与障碍物的距离设计reward 让leader选择一条能使整支编队绕开的路线  
因为leader必须知道followers的位置和formation size 所以leader与followers需要bidirectional communication  
作者仍称其为distributed architecture 但从obstacle decision角度看 它实际上更接近leader-centralized avoidance  
这种方法减少了followers的sonar需求 但强依赖leader感知覆盖、通信和followers的formation-tracking精度  

#### 动力学模型
仿真对象使用REMUS参数的nonlinear 3-DoF horizontal dynamics：  
state为position/heading [x,y,psi]和body velocity [u,v,r]  
模型包含surge、sway、yaw方向的added mass、hydrodynamic damping、cross-coupling、body/fin lift以及rudder作用  
control input是propeller thrust X_prop和rudder deflection delta_r  

这里必须注意：这篇虽然研究多艇编队 但每艘艇只有3-DoF模型并保持constant depth  
没有z/heave、roll和pitch运动 也没有三维编队或上下绕障  
因此不能因为对象是AUV就把它理解成multi-AUV 6-DoF formation control  

#### Observation / state space
每一艘AUV分别有自己的MDP和TD3 agent  

leader observation主要包括：  
leader heading相对goal line-of-sight angle的误差  
body velocity [u,v,r]  
传感器范围内各个identified obstacle的scaled distance vector  

follower observation主要包括：  
相对leader的normalized distance error  
相对desired formation angle的angle error  
leader和follower heading  
leader/follower speed以及desired formation speed  
Approach 1中还包括identified obstacle distances  
Approach 2中followers完全没有obstacle input  

论文列举IMU、compass、DVL、depth sensor、GPS surface correction、sonar和communication modem作为可能的真实传感器  
但这些只是概念性的system architecture 实际神经网络没有读取raw IMU/DVL/sonar message  
仿真直接向policy提供heading、velocity、relative formation error以及到每个障碍物的精确距离  

这里仍有Hadi 2022工作相同的fixed-input问题：obstacle vector写成N维 而MLP需要固定输入dimension  
论文没有充分解释障碍物数量变化时如何padding、排序、截断或编码 因而实际部署与复现存在缺口  

#### Action space
每艘AUV的TD3 action是连续二维向量：  
a = [X_prop, delta_r]  
X_prop范围是3–13 N  
delta_r范围是-20 deg到20 deg  

所以这次RL同时控制forward propulsion和heading：  
X_prop调节前进速度  
delta_r通过rudder改变yaw/航向  

它比Hadi 2022仅输出rudder angle的动作空间更完整 但X_prop仍是理想propeller thrust command 不是motor RPM、voltage或PWM  
论文没有进一步建模motor/propeller transient、dead zone、thrust saturation error和control allocation  

#### Reward
leader reward包含：  
target-heading reward：惩罚leader heading偏离当前goal LOS angle  
obstacle avoidance：进入安全距离后按距离惩罚  
self-collision avoidance：AUV之间小于安全距离时惩罚  
control effort：惩罚propeller thrust和rudder magnitude  

follower reward包含：  
formation distance error  
formation angle error  
obstacle avoidance（只用于Approach 1）  
self-collision avoidance  
control effort  

值得注意的是 leader所谓target reward实际主要惩罚heading error 不是直接的distance-to-goal progress或path length  
episode以leader进入goal region作为成功终止条件 但reward公式并没有严格优化全局最短路  
因此论文所说shortest/optimal path更多是由朝向目标、避障和动作惩罚间接诱导 不是显式最短路径保证  

#### TD3与训练
每个agent使用独立TD3 actor-critic和独立experience replay  
每个TD3包含online/target actor以及两组online/target critics 即每个agent共2个actor networks和4个critic networks  
actor和critic使用两层fully connected MLP hidden sizes为400和300 ReLU activation  
exploration使用Ornstein-Uhlenbeck noise  
actor learning rate 0.001 critic learning rate 0.0001 discount factor 0.99 replay memory 1e6 soft update 0.005  
每个episode最多300 steps 图中训练到5000 episodes  
训练硬件是AMD Ryzen 7 3800XT、NVIDIA RTX 2060 在MATLAB中运行  
论文没有报告wall-clock training time  

论文给出的sample time为0.1 s 但300 steps只相当于30 s  
从训练区中心到边界约需数百米 以1.5 m/s运动通常远超过30 s 而结果曲线也延伸到约150 s  
因此paper中的episode length、sample time或绘图时间尺度至少有一项没有解释一致 这是复现时需要核对的问题  

#### 仿真环境和实验
训练区域为500 m x 500 m square  
三艘AUV初始位于中央区域 leader初始位置为(250,250) followers为(220,220)和(280,220) 初始heading为0  
每艘1 m长的AUV用radius 0.5 m circle近似碰撞体  
目标放在区域边界并在episode中随机变化  
障碍物被建模为随机放置的圆形几何体 论文采用14 m的障碍物半径设置  
desired formation speed为1.5 m/s  

测试包括：  
没有障碍物时到达不同目标并建立三角编队  
不同相对位置的单个/多个静态障碍物  
障碍物进入formation内部  
两个障碍物间距小于formation width时followers分别绕行再恢复编队  
Approach 1与Approach 2的编队避障示例  
随机ocean-current speed 0–0.3 m/s direction 80–140 deg  
加入communication delay和navigation errors后的单个trajectory comparison  

#### 海流、通信延迟和导航误差
海流通过relative velocity v_r = v - v_c进入3-DoF hydrodynamics  
作者用Markov moving-average process随机改变流速和方向  

通信延迟用Rayleigh-distribution time-varying delay模拟 论文选择的示例峰值约0.1 s  
navigation error同样用Markov moving-average process加到x/y/heading状态 位置误差图大约在数米范围内变化  
这些扰动没有加入training 只在trained policy测试阶段加入 这能说明policy对所选扰动样例具有一定容忍度  

但这不等于已经解决真实underwater networking：  
没有模拟packet loss、有限bandwidth、acoustic update rate、out-of-order messages、通信range和网络拥塞  
也没有说明followers收到的是延迟状态后如何timestamp、prediction或state estimation  
结果主要是少量trajectory plots 没有大规模Monte-Carlo success rate或delay/error sensitivity curve  

#### 是否有真实机器人
没有 全部结果是MATLAB computer-based simulations  
没有水池实验、海试、hardware-in-the-loop、真实多AUV通信或真实sonar input  

论文Implementation Remarks中讨论Jetson Nano/TX2理论上可以运行这些网络 并引用其他论文的硬件实现  
但那不是本文自己的hardware experiment  
作者明确把real-time implementation描述为下一步工作 不能把“implementation is feasible”理解成“已经实现”  
论文没有提供明确的官方代码/GitHub链接 当前未确认开源  

#### “unknown environment”和“end-to-end”的真实含义
unknown environment表示policy事先没有固定的global obstacle map 并在episode中面对随机障碍物  
但进入sensor range后 simulator直接提供每个障碍物的准确距离 并且碰撞体半径和安全距离都由reward已知  
它没有解决raw sonar echo -> obstacle detection/tracking -> relative geometry这一感知过程  

end-to-end表示：  
structured AUV/formation/obstacle state -> propeller thrust + rudder angle  

并不是：  
raw sonar/IMU/DVL/acoustic messages -> localization/data association -> multi-AUV action  

因此它是privileged/idealized-state下的end-to-end control 而不是真实传感器意义上的multi-AUV autonomous navigation  

#### 创新点
1）把goal reaching、formation maintenance、environmental obstacle avoidance和inter-AUV collision avoidance放进同一套multi-agent TD3框架 而不是分别设计planner和formation controller  
2）让RL直接输出continuous propeller thrust和rudder angle 同时控制speed和heading  
3）提出两种具有不同sensor/communication成本的避障结构：all-agent local avoidance与leader-only whole-formation avoidance  
4）针对leader和followers设计不同的state/reward 并测试海流、通信延迟和导航误差  

真正有意思的贡献不是TD3算法本身 而是两种formation avoidance information architecture之间的取舍：  
Approach 1传感器成本高、通信依赖较低、局部安全性更好  
Approach 2传感器成本低 但更依赖leader感知、双向通信和精确formation state  

#### 相对Hadi 2022 single-AUV工作的进步与退步
进步：  
从single AUV扩展到leader + two followers  
从只控制rudder扩展为同时控制propeller thrust和rudder  
加入formation error、self-collision和inter-agent communication问题  
提出两种避障/信息交换架构  

退步或进一步简化：  
Hadi 2022底层使用6-DoF REMUS dynamics 虽然navigation task是2D 这篇直接降成3-DoF horizontal model  
仍然只处理constant depth和static circular obstacles  
仍然直接使用精确obstacle distance与position/heading 没有真实感知和定位pipeline  
没有实物验证 也没有与成熟formation controller/planner或其他MARL方法做充分benchmark  

因此这篇不能简单理解成“Hadi 2022工作的全面升级版”  
它主要在multi-agent task formulation和reward/information architecture上前进 但physical realism和experimental validation没有同步提升  

#### 主要limitations
只测试3艘AUV和固定三角formation 虽然作者说followers可以继续作为leader扩展队形 但没有实际验证larger swarm scalability  
每增加一艘AUV都增加一套actor/critics/replay buffer 通信量和训练成本如何增长没有分析  
obstacle-distance vector长度N可能变化 与固定MLP输入不兼容 论文没有说明encoding方法  
障碍物是静态圆形几何体 没有动态障碍、非规则结构、狭窄通道或三维上下避障  
Approach 2假设leader能感知足够早 且followers能准确跟随；如果障碍物只出现在follower局部视野中 leader-only sonar可能失效  
circumscribed-circle方法把formation视为刚性整体 可能过度保守 也可能在formation transient或通信延迟下失真  
通信/导航扰动模型较简单 没有真实acoustic modem和定位数据验证  
reward依赖较多人工权重 没有系统ablation或与经典distributed MPC/backstepping等方法比较  
只展示少量trajectory和error curves 缺少success rate、collision rate、path efficiency、energy和统计置信区间  
训练时间、部分时间尺度和输入维度细节不足 复现性有限  

#### 对我们后续水下导航/多机器人研究的启发
这篇最值得保留的是“导航policy和formation policy如何分工”的问题 而不是直接照搬其理想distance input  
如果未来做multi-AUV navigation 可以把结构设计为：  
单艇local safety policy：根据真实sonar/history输出safe local velocity 任何时候具有最高安全优先级  
formation policy：根据延迟的neighbor relative state生成formation correction  
leader/global guidance：只提供goal/path intent 不直接覆盖单艇紧急避障  

这样相当于吸收Approach 1的local safety优点 同时避免每个agent完全独立学习造成的冲突  
训练时可用perfect obstacle/neighbor states作为privileged teacher 部署时student读取真实sonar、DVL/IMU和acoustic-message history  
通信输入应显式包含message age/timestamp 并用RNN/transformer处理delay和packet loss 而不只是给状态加一个随机延迟  
评价时至少报告不同队伍规模、障碍密度、通信丢包率、定位漂移和海流强度下的success/collision/formation error/energy  

#### Path-Following Control of Unmanned Underwater Vehicle Based on an Improved TD3 Deep Reinforcement Learning
2024 University of Warwick  
作者：Yexin Fan / Hongyang Dong / Xiaowei Zhao / Petr Denissenko  
发表/收录：IEEE Transactions on Control Systems Technology Vol.32 No.5 pp.1904–1919；online publication 2024-03-27，正式卷期为September 2024  
DOI：https://doi.org/10.1109/TCST.2024.3377876  

这篇提出TD3-IMP 用改进TD3控制BlueROV2沿预先给定的二维或三维路径运动  
它的主要目标不是让RL寻找一条从起点到目标的无碰撞路径 而是在路径已经给定后 尽量减小横向误差、深度误差和yaw误差  
系统里仍然保留传统LOS guidance：LOS根据几何路径计算desired heading和path error TD3只负责根据这些误差和当前速度生成连续控制量  

因此更准确的系统层次是：  
predefined path / waypoints -> LOS guidance -> tracking errors + desired heading -> TD3-IMP -> 4-DoF channel commands -> thruster allocation / PWM  

它位于高层navigation和纯低层定点控制之间 更像learning-based path-following controller  
它不包含camera/sonar perception、obstacle detection、collision avoidance、global planning或unknown-environment exploration  

##### 任务和能力边界
论文从通用6-DoF UUV动力学开始 但针对BlueROV2实际只控制4个DoF：  
surge  
sway  
heave  
yaw  

roll和pitch不进入policy控制 作者假设通过浮心/重心配置提供机械静稳性 因而phi和theta保持接近0  
这不是任意6-DoF姿态控制 也不是Learning to Swim那种完整position + orientation regulation  

训练路径是一条3D helix  
x_r = 2 - 2cos(zeta)  
y_r = 2sin(zeta)  
z_r = -2zeta - 0.5  
zeta in [0,2pi]  

训练完成后 policy直接测试没有见过的sine、comb scanning、closed curve、figure-eight和另一条更长的3D helix  
这里的generalization是对不同path geometry的泛化 不是对新障碍、新场景图像或真实传感器分布的导航泛化  

##### 动力学仿真和domain randomization
仿真器基于标准Fossen式6-DoF动力学：  
M v_dot + C(v)v + D(v)v + g(eta) = tau + Delta_tau  
eta_dot = J(eta)v  

模型包含inertia / added mass、Coriolis-centripetal、hydrodynamic damping、hydrostatic force以及外部force/moment  
BlueROV2基准水动力参数引用已有文献 推力关系则采用经过实验验证的linear approximation  
所以RL算法本身可以称model-free 但训练环境当然不是“无模型” 它依赖人工建立的BlueROV2动力学仿真器  

训练时把hydrodynamic parameters在nominal value的正负20%范围内随机采样  
同时加入三个方向的sinusoidal wave force：  
Delta_tau_x = 6 rand(-1,1) sin(0.1t) N  
Delta_tau_y = 5 rand(-1,1) cos(0.2t) N  
Delta_tau_z = 4 rand(-1,1) cos(0.3t) N  

这里的disturbance仍然相当理想化：只有预设频率的正弦平移力 没有随机海流场、波浪谱、涡流、yaw moment、thruster dynamics或传感器延迟  
后面的仿真抗扰测试使用相同形式和幅值的正弦扰动 因而主要是训练分布内的robustness test 不能等同于对任意未知海洋扰动的泛化  

##### LOS guidance和observation
水平面LOS根据当前path segment P_(k-1)P_k、cross-track error e_d、path slope gamma_p和look-ahead distance L产生desired yaw：  
psi_d = gamma_p + arctan(-e_d/L)  

垂直面同样用projection / P_LOS计算depth error e_z  
所以policy不是自己从waypoints推断应当往哪里走 LOS已经把几何路径转换成适合控制器跟踪的误差信号  

observation共11维：  
S = {e_l, e_d, e_z, u, v, w, e_delta, sin(psi_d), cos(psi_d), sin(psi), cos(psi)}  

e_d是horizontal cross-track error  
e_z是depth error  
u/v/w是body-frame linear velocity  
e_delta = psi - psi_d是heading error  
psi_d和psi同时用sin/cos编码 避免角度在正负pi处不连续  

论文写e_l = L 即look-ahead distance 这意味着该维可能是常量 而reward里又把e_l放进distance norm  
作者没有解释L是否会随path segment或状态变化 因而这一维的实际信息量和公式实现存在歧义  
论文还在不同公式中交替使用e_delta和e_psi表示heading error 是另一个符号不一致之处  

policy没有读取raw IMU、camera、DVL、sonar或RTK measurement  
部署前必须先由定位和EKF把传感器数据转换成position、velocity、yaw和path error 这仍然是structured-state controller 不是sensor-to-motor end-to-end policy  

##### Action和thruster allocation
TD3 actor输出连续4维action：  
A = {a_x, a_y, a_z, a_yaw}  
分别对应surge / sway / heave / yaw channel control command  
actor最后一层使用tanh 每个通道归一化到[-1,1]  

它不是离散动作 也不是只控制方向舵  
它也不是直接独立输出BlueROV2每个推进器的RPM/PWM  
论文先输出四个运动通道命令 再通过thrust configuration / allocation关系转换成六个thruster commands 最终由ESC执行PWM  

论文把tau = H T写成thruster command到body wrench的关系 但在实验描述中又说用H把channel command转换成individual thruster command  
严格实现通常需要allocation inverse / pseudoinverse 文中没有把这一方向和饱和处理写清楚  

##### Reward和dynamic constraint
总reward为：  
r = r_e + r_psi + r_rho  

r_e是distance/path-error reward 由e_d、e_l和e_z的norm构成 指数形式鼓励靠近目标路径 另外减去常数c 防止机器人停着不动也能避免惩罚  
r_psi根据heading error奖励方向对准 并对超过正负90度的朝向给更强惩罚  
r_rho是额外的dynamic constraint reward 用一个以path projection point为中心的球形容许区域约束探索  

这个容许半径rho_n不是固定值：  
训练早期recent episode reward较低时 rho_n约为10m 允许较宽松探索  
训练表现改善后半径逐步缩小  
reward ratio达到80%后半径缩到1m 迫使policy进行更精确的path tracking  

可以把它理解成基于训练表现的automatic curriculum：先教机器人不要完全跑丢 再逐渐提高精度要求  
但它依赖手工设置rho_min = 1m、rho_max = 10m以及recent reward ratio阈值 论文没有单独做移除r_rho的ablation 因而无法从实验中严格量化这部分贡献  

##### TD3-IMP的三个改进
第一项是improved experience replay  
作者同时维护normal transition buffer M_T和high-reward episode buffer M_R  
M_T保存一般transition 并按absolute TD error进行PER sampling 让当前critic预测错误较大的样本更容易被抽到  
M_R保留average episode return较高的整段trajectory 当新episode表现优于buffer里最差episode时 用新trajectory替换它  
每次mini-batch按比例混合两类样本：一部分来自TD-error prioritized buffer 一部分来自high-return trajectory buffer  

它的思想是同时保留两种“重要经验”：  
TD error大 -> 对当前value function新颖或尚未学好  
episode return高 -> 已经展示了较好path-following行为  

相比普通PER只追逐TD error 这种方法希望兼顾exploration和对成功轨迹的利用  
不过论文没有给出reward buffer容量按episode还是transition严格管理的全部细节 Algorithm中的xi采样方向、buffer初始化和正文表述也不够完整 复现时仍需自行判断  

第二项是adaptive action smoothness  
作者基于CAPS在actor objective里加入temporal smoothness和spatial smoothness：  
相邻time step的state应该输出相近action  
对当前state加入小扰动后也应该输出相近action  

关键变化是temporal smoothness weight lambda_T不是固定常数：  
lambda_T = k_D1 exp[-k_D2(|e_d| + |e_delta| + |e_z|)]  

tracking error大时lambda_T变小 允许policy快速、大幅修正动作  
tracking error小时lambda_T变大 抑制不必要的高频thruster oscillation  
因此它试图在tracking accuracy与actuator smoothness之间动态折中 不是简单对输出做low-pass filter 也不会在部署后额外改变policy dynamics  

第三项是前面提到的dynamic reward radius  
训练早期使用宽松约束 后期随着表现提升自动收紧 用来减少无效探索并加快收敛  

##### 网络和训练
算法使用TD3的standard twin-critic / delayed actor update / target policy smoothing / soft target update结构  
actor和critic均为两层MLP 每层256 neurons  
actor learning rate 0.0003 critic learning rate 0.001 discount factor 0.99 batch size 128  
normal buffer M_T容量40000 reward buffer M_R容量200000  

训练700 episodes 每个episode最多400 steps  
初始位置在reference helix起点附近随机化 yaw从(-pi,pi)随机采样  
如果离路径超过10m或达到最大steps episode终止  
环境是Ubuntu 18.04 + PyTorch 1.7  

论文没有报告simulation time step、control frequency、CPU/GPU型号、wall-clock training time或代码仓库  
所以它证明的是episode-level sample efficiency提升 不能和Learning to Swim / Sim2Swim的分钟级GPU并行训练速度直接比较  

##### Simulation results
训练曲线比较DDPG、vanilla TD3、TD3-PER和TD3-IMP 每条曲线来自5次trial的mean和standard deviation  
TD3-IMP大约150 episodes达到接近最高reward  
TD3-PER约300 episodes TD3约400 episodes DDPG约450 episodes才达到相近水平  
说明high-return replay + dynamic reward确实与更快的episode convergence相关 但论文没有逐项拆开这两个设计做完整factorial ablation  

action-smoothness实验比较固定lambda_T = 0.005、固定0.02和dynamic lambda_T：  
固定0.005 tracking较准但thruster command明显振荡  
固定0.02最平滑但过度限制转向 导致sharp turn失败  
dynamic lambda_T的horizontal/depth/yaw MAE为0.211m / 0.158m / 0.092rad smoothness metric为0.0209%  
它比小lambda平滑很多 同时没有大lambda的严重tracking degradation  

unseen 2D path测试中 TD3-IMP与PID的MAE分别为：  
sine 0.012m vs 0.015m  
comb 0.018m vs 0.029m  
closed curve 0.025m vs 0.183m  
figure-eight 0.029m vs 0.593m  

但PID只在sine path上fine-tune一次 然后原参数直接用于更复杂路径  
这能展示TD3-IMP不重新调参的便利 但不能证明它优于每条path都重新整定的PID 更没有与MPC、adaptive control或现代robust controller公平比较  

在正负20% hydrodynamic uncertainty的3D helix测试中重复15次 TD3-IMP全部完成 PID失败2次  
在sinusoidal disturbance下 TD3-IMP轨迹比PID平滑 horizontal/depth error更小  
不过测试扰动形式与训练时相同 因而鲁棒性证据仍局限在作者设定的模型族内  

##### Real-world experiments
真实平台是六推进器BlueROV2 测试地点为深度超过30m的湖  
硬件包括IMU、depth sensor、STM32、Raspberry Pi、RTK-GNSS module以及与host computer的tether/Ethernet connection  
sensor measurements通过EKF融合  

由于GPS/RTK不能在水下工作 作者没有完成一次由同一定位系统闭环的完整3D path-following实验  
论文明确把实物验证拆成两部分：  
1）underwater depth tracking  
2）water-surface horizontal path following  

Depth tracking让机器人分别跟踪-6m、-10m和-20m setpoint 论文报告MAE为0.02m、0.06m和0.05m  
这证明heave/depth channel能够工作 但不是同时跟踪x/y/z的3D curve  
论文说下降过程中的小波动来自机器人碰到underwater rocks 这反而说明实验没有obstacle avoidance或independent safety layer 也使这组试验的安全设计和可重复性值得谨慎看待  

Surface path由坐标(0,0)、(10,0)、(10,15)、(20,15)、(30,0)构成  
作者分别在wind level 2和wind level 5下比较TD3-IMP与PID  
轻微风浪时两者horizontal MAE都是0.05m yaw error都是0.1rad  
wind level 5时TD3-IMP约0.13m / 0.2rad PID约0.25m / 0.4rad  

这些结果支持TD3-IMP在该次surface-wave条件下比固定PID更稳健  
但wind level只是定性环境标签 没有实测current、wave spectrum、force magnitude或多次trial统计  
坐标列表实际只有5个waypoints 通常只能连接出4条segment 而正文称5 straight lines + 5 waypoints 也存在描述不一致  

##### 是否zero-shot sim-to-real
不是  

在实物实验前 作者先驾驶UUV在真实水环境采集state-action-transition data  
然后把真实数据与simulation data合并进experience replay buffer 再重新训练neural networks  
最后保存actor weights并部署到真实path-following experiment  

因此更准确的分类是real-data-assisted sim-to-real / mixed simulation-real replay retraining  
它与DDR都需要目标机器人数据 但适配位置不同：  
DDR保持source SAC policy 主要学习simulation action到real action的adapter  
这篇直接把真实transition混进replay buffer并重新训练TD3 policy  

论文没有报告真实数据量、采集时长、采集controller、simulation/real sampling ratio、retraining episodes以及是否更新所有actor/critic networks  
这是整篇最重要的复现缺口之一 也意味着不能把实物结果归因于单纯的正负20% domain randomization  

##### 主要limitations
任务是given-path tracking 不是自主navigation policy 路径与LOS guidance都由外部提供  
没有camera/sonar输入、环境理解、obstacle avoidance或动态障碍处理  
只控制4-DoF roll/pitch依赖被动稳定 不能控制任意orientation  
真实验证被拆成surface x-y tracking和underwater z tracking 没有真正同时验证3D helix path  
需要真实数据retraining 不是zero-shot 部署流程细节和数据量未报告  
surface实验依赖RTK-GNSS 水下depth实验缺少全局horizontal positioning 不能说明GPS-denied 3D localization能力  
系统仍有tether/Ethernet和host computer 没有证明untethered onboard autonomy  
真实扰动用wind level定性描述 缺少重复试验、error bar和扰动测量  
PID baseline没有为每种路径重新调参 比较有利于RL 且缺少MPC/SMC/adaptive controller实物baseline  
动力学随机化和wave model较简单 没有thruster failure、current field、sensor noise/latency、packet loss或allocation uncertainty  
paper中的e_l/L、e_delta/e_psi、allocation matrix方向以及waypoint/segment数量有若干符号或描述歧义  
没有公开代码、训练时间和完整部署参数 复现性一般  

##### 放在领域时间线里的位置
与2022 Hadi single-AUV TD3 navigation相比：  
Hadi做random goal reaching + static obstacle avoidance action只有rudder且forward speed基本预设  
Fan 2024不做障碍/目标搜索 而是跟踪给定path 但action扩展到surge/sway/heave/yaw四个连续通道 仿真从2D任务推进到3D path并增加了湖上实物验证  

与2023 DDR相比：  
两者都不是zero-shot 都使用少量真实数据缓解sim-to-real gap  
DDR只学习快速action mapping用于position regulation Fan则把真实transition回灌replay并重新训练完整path-following policy  

与2023/2024 Hadi formation work相比：  
formation paper同时控制propeller thrust和rudder 但仍是2D constant-depth multi-AUV simulation  
Fan回到single UUV 去掉formation与obstacle avoidance 换来4-DoF path tracking、动作平滑设计和部分实物验证  

与2025 Learning to Swim相比：  
Fan通过LOS path error输出4个channel commands 部署前需要真实数据retraining  
Learning to Swim直接做6-DoF position/orientation regulation 输出individual thruster commands 并通过massively parallel simulation + domain randomization实现zero-shot pool transfer  
Fan的优势是明确研究了experience replay效率和action oscillation；Learning to Swim的优势是完整6-DoF、训练并行化和更清楚的zero-shot定位  

从领域发展看 这篇代表一个过渡阶段：  
早期RL-AUV工作主要在MATLAB动力学里做2D heading / goal reaching  
Fan 2024开始把continuous DRL扩展到4-DoF path-following 并认真处理action smoothness和真实部署  
但sim-to-real仍依靠真实数据回灌 定位和guidance仍是传统模块  
后续Learning to Swim / Sim2Swim则进一步转向GPU massive parallelization、6-DoF control和zero-shot deployment  

##### 对我们的启发
如果目标是水下自主导航 不应该直接把这篇TD3-IMP当作完整navigation solution  
更合理的复用方式是把它放在系统中层：  
perception / mapping / planner生成safe path  
LOS或其他guidance把path变成desired velocity / heading  
learning controller负责4-DoF或6-DoF tracking  
independent safety layer处理sonar obstacle和actuator limits  

它最值得借鉴的两个技术点是：  
1）根据tracking error动态改变action smoothness：误差大时允许快速修正 误差小时保护thruster并减少能耗  
2）同时replay高TD-error transition和high-return whole trajectory：兼顾尚未学好的状态与已经成功的控制片段  

如果继续做研究 更严谨的下一步应是：  
用真实DVL/INS/pressure/sonar形成GPS-denied 3D state estimate  
在同一次实验中同时跟踪x/y/z/yaw而不是拆分surface和depth  
加入obstacle-aware high-level policy或安全过滤器  
把sim/real mixed replay与domain randomization、action adapter、zero-shot baseline做消融  
报告真实数据量、训练时间、多次trial统计、energy和thruster wear指标  

论文公开accepted manuscript：papers/underwater/Fan_2024_Improved_TD3_UUV_Path_Following.pdf  
IEEE：https://ieeexplore.ieee.org/document/10480708  

#### Toward 6-DOF AUV Energy-Aware Position Control based on Deep Reinforcement Learning
2024/2025 MBARI / Pontificia Universidad Catolica de Chile  
发表/状态：列入2024 IEEE/OES Autonomous Underwater Vehicles Symposium会议日程；扩展的arXiv版本于2025-02公开  
链接：https://arxiv.org/abs/2502.17742  

这是Swim4Real的前期版本，不应再当成一篇与Swim4Real平行的最终成果。  
它已经确定了后续工作的核心路线：用TQC做6-DoF目标位姿调节，policy直接输出各推进器命令，并在reward里惩罚动作幅值以降低能耗。  
但该版本只做Stonefish仿真，没有真实机器人验证；Swim4Real在它的基础上增加absolute attitude观测、更多目标姿态、domain-randomized policy、系统仿真评估和水槽实物实验。  

#### Learning to Swim: Reinforcement Learning for 6-DOF Control of Thruster-driven Autonomous Underwater Vehicles
2024 WHOI/MIT相关方向  
发表/收录：ICRA 2025（IEEE International Conference on Robotics and Automation） arXiv版本写“To appear at ICRA 2025”  
这篇更偏低层运动控制 而不是视觉导航  
核心任务是 full 6-DOF position/orientation control：给AUV一个期望6-DoF pose 让它通过thruster直接控制自己到达/保持该pose  
它不处理图像 不做导航规划 不做避障 也不输出高层yaw/pitch steering 而是把目标pose和当前state直接映射到6个thruster commands  
和Nav2Goal互补：Nav2Goal是图像+goal输出离散yaw/pitch高层动作；Learning to Swim是状态目标到thruster的低层6-DoF控制  

#### 任务和方法能力
目标是训练一个command-conditioned 6-DOF controller  
输入是相对目标位置/姿态和当前机器人状态 输出是6个推进器命令  
同一个policy可以用于position holding 也可以用于trajectory following 因为它学的是把local-frame position offset驱动到0  
真实验证主要是水池里的position hold + disturbance rejection：AUV保持在AprilTag上方 人为用棍子推它 看它能不能回到目标pose  
所以这篇证明的是“RL thruster-level controller可以zero-shot真实部署 并能抗扰动” 还不是复杂任务级水下导航  

#### 仿真器
作者实现了一个GPU并行水下AUV仿真器 代码叫isaac-auv-env 基于NVIDIA Isaac Lab / Isaac Gym系框架  
目标是像腿式机器人那样一次并行跑上千个环境 用PPO快速训练水下控制器  
他们可以并行2048个环境 在A6000上大概用11.4GB显存 总训练时间约10-20分钟  
这篇的一个重要贡献就是：把简化水动力模型接进Isaac Lab 让水下控制也能用massively parallel RL  

仿真水动力模型比较简化：  
drag / viscous resistance 用MuJoCo里的simplified inertia-based hydrodynamics model 把AUV近似成矩形体  
buoyancy根据估计volume和center of buoyancy计算  
thruster dynamics用Yoerger模型 Thrust(Ω)=Ct||Ω||Ω  
推进器型号参考BlueRobotics T200 真实机器人CUREE也用6个T200  
仿真参数包括water density viscosity thruster rotor constant CoB-CoM offset volume mass等  

#### Domain randomization
水下sim-to-real里最重要的不确定性之一是payload变化导致的体积/浮心/质心变化  
作者主要randomize两个东西：  
CoB-CoM offset noise（center of buoyancy和center of mass偏移）  
Volume noise（体积变化 影响浮力）  
设置了No DR Small DR Large DR三种范围  
结论是Small DR最好：有一定randomization能提高domain transfer 但Large DR会让policy变得过于保守 因为它学的是跨很多动力学的平均行为  

#### 观测和输出
policy是2-layer MLP 输入维度17  
观测 o = {x_offset, q_des, q, x_dot, omega}  
x_offset：期望位置和当前位置的差 在AUV local frame下表达  
q_des：期望姿态四元数  
q：当前姿态四元数  
x_dot：当前线速度  
omega：当前角速度  
注意这里用的是状态估计 不用图像特征  

动作输出 a 是6个thruster commands 归一化到[-1,1] 再scale到PWM  
也就是说它是direct-to-thruster controller 不是PID外面套一个高层policy  
真实部署时policy接收EKF state estimate和用户/轨迹给的desired pose 输出thruster commands 约20Hz运行  

#### reward
每步正奖励由三部分组成：  
position reward：rx = exp(-||x_offset||^2) 鼓励位置到目标  
orientation reward：rq = exp(-|angle(q_des, q)|) 鼓励姿态对齐 用四元数差的axis-angle角度  
energy/action reward：rp = exp(-||a||^2) 鼓励少用推进器/低能耗  
总reward是 rt = λx rx + λq rq + λp rp  
每个训练episode只有3秒 experience time 初始位置偏差最多2m 姿态目标随机  

#### 训练和真实部署
训练算法用RSL-RL里的PPO（Proximal Policy Optimization）  
训练时环境随机初始化unit rotations和position offsets 目标姿态也随机  
在仿真里评估domain transfer：nominal环境和shifted dynamics环境 shifted环境里CoB-CoM前移0.2m volume降低1.5L 让AUV更负浮力  
Small DR在shifted dynamics下误差明显小于No DR 说明对物理参数变化更鲁棒  

真实机器人是CUREE AUV 6个BlueRobotics T200 thrusters + Jetson Orin NX  
状态估计来自onboard cameras检测AprilTag + DVL（Doppler Velocity Logger）+ IMU 经EKF融合  
policy zero-shot部署 没有真实数据训练/finetune  
实验是position hold disturbance rejection：机器人保持目标pose 人用棍子强推扰动 它能回到目标  
和手调PID相比 RL controller更aggressive 但有steady-state error 尤其pitch和y方向  

#### limitation / future work
能力边界：这篇是低层6-DoF控制器 不是完整自主导航系统  
没有视觉导航 没有障碍物 没有路径规划 没有海流显式估计  
真实实验是小水池position hold + external push 不是真实海洋长距离任务  
simulator的水动力模型很简化 没有建模motor action delay 作者说未建模delay可能导致部署不稳定 所以用了quaternion slerp平滑姿态目标  
domain randomization只能学到平均鲁棒行为 不能根据当前payload/水动力实时自适应 所以会出现steady-state error  
部署中还发现major thruster imbalance / 非流线型外壳导致的非对称推进性能 这些不在模型里  
future work：online learning / adaptation 让控制器能在测试时适应新物理情况 减少steady-state error  
另一个future direction是加入更多类型hydrodynamics model 支持非thruster类推进方式 因为这些推进方式非线性更强 更难控制  

#### 对我们的启发
如果我们研究水下learning-based motion control 这篇比Nav2Goal更关键  
它说明水下控制可以借鉴腿式机器人massively parallel RL的范式：并行仿真 + PPO + domain randomization + zero-shot sim-to-real  
但它也说明仅靠DR还不够 水下payload/浮力/推进器不平衡会带来明显steady-state error 之后可能需要online adaptation / latent dynamics estimation  
从系统角度 可以把Nav2Goal看作高层视觉导航 把Learning to Swim看作底层6-DoF控制 一个完整系统可能是：视觉/声呐导航policy给desired pose或velocity 然后Learning-to-Swim式controller输出thrusters  
链接：https://arxiv.org/abs/2410.00120

#### Swim4Real: Deep Reinforcement Learning-Based Energy-Efficient and Agile 6-DOF Control for Underwater Vehicles
2025 MBARI / Pontificia Universidad Catolica de Chile  
作者：Vicente Sufan / Giancarlo Troni  
发表/收录：IEEE Robotics and Automation Letters, Vol.10, No.7, pp.7326-7333, July 2025  
DOI：https://doi.org/10.1109/LRA.2025.3575650  
IEEE：https://ieeexplore.ieee.org/document/11020757/  
公开accepted manuscript：https://www.researchgate.net/publication/392344314_Swim4Real_Deep_Reinforcement_Learning-Based_Energy-Efficient_and_Agile_6-DOF_Control_for_Underwater_Vehicles  

这篇是前面Toward 6-DOF Energy-Aware工作的正式扩展版。  
论文标题里有agile，正文也把系统称为end-to-end controller，但任务边界要说得非常准确：  
它做的是**6-DoF fixed-pose setpoint regulation**，也就是从随机初始位置和姿态移动到一个固定目标位置与目标姿态并稳定下来。  
它不是跟踪随时间变化的trajectory，也不做path planning、waypoint navigation、obstacle avoidance或camera/sonar perception。  

##### 任务与系统层次
被控平台是MBARI的MOLA AUV，一台全驱动holonomic机器人：  
尺寸约0.72m x 0.45m x 0.39m  
具有8个三维矢量布置的推进器  
传感器包括IMU、DVL、depth sensor、camera和sonar  

但camera和sonar没有进入RL policy。  
真实闭环使用IMU、DVL与深度信息形成6-DoF state estimate，再把当前状态与目标位姿的误差交给policy。  

因此完整接口是：  
desired 6-DoF pose + estimated current state -> TQC policy -> 8 normalized PWM commands -> individual thrusters  

这里的end-to-end仅表示从结构化状态/目标直接输出各推进器命令，中间不经过显式PID或thrust allocation。  
它并不是raw camera / sonar / IMU packet到motor的感知端到端系统。  

##### Observation和Action
observation是23维并归一化到[-1,1]：  
1）6维pose error：3维位置误差 + 3维姿态误差  
2）3维absolute attitude：roll / pitch / heading  
3）6维twist：3维线速度 + 3维角速度  
4）8维previous action  

previous action相当于给无记忆MLP一个单步动作历史，使policy知道当前推进器之前施加了什么命令，也有助于减少指令突变。  
absolute attitude是相对前期Toward论文新增的观测；它让policy在相同pose error下仍能区分机器人在世界坐标系里的实际朝向，而这会改变浮力恢复力和不同方向的运动代价。  

action是8维连续量，每一维属于[-1,1]，直接表示MOLA八个推进器的normalized PWM。  
所以它与Sim2Swim不同：  
Swim4Real直接学习8个individual thruster commands  
Sim2Swim输出6维body force / torque，再由传统thrust allocation分给推进器  

作者说方法不需要先验thruster configuration，准确含义是policy不需要显式给定allocation matrix。  
但8维输出及其训练动力学仍然绑定MOLA的推进器数量、位置、方向和推力响应；换成另一条艇不能期待原policy直接通用，通常需要重新建模和训练。  

##### TQC算法
作者使用TQC（Truncated Quantile Critics），它是off-policy continuous-control算法。  
critic不只预测一个期望Q值，而是用多个quantile估计return distribution；更新target时丢弃最高的一部分quantiles，以降低连续控制里常见的Q-value overestimation。  
同时保留SAC式的随机actor和entropy regularization，使训练能探索不同推进器组合。  

使用Stable-Baselines3实现。主要超参数为：  
learning rate 0.003  
replay buffer 1,000,000  
batch size 256  
discount factor 0.99  
2个critic，每个25个quantiles  
actor / critic MLP hidden layers为512 / 512 / 256  

论文同时训练两个版本：  
REEF：固定nominal dynamics  
REEF-DR：每个episode随机质量的domain-randomized版本  

REEF是作者给控制器起的名字，不是另一个算法；两者都使用TQC。  

##### Reward与“energy-aware”的真实含义
reward包含四类目标：  
位置误差  
姿态角误差  
相邻两步action变化  
当前action幅值  

其中position使用各轴absolute error，orientation使用姿态旋转误差；后两项分别抑制推进器抖动和持续大推力。  
论文手工设置的六个权重为：x位置误差alpha1=-4、y位置误差alpha2=-4、z位置误差alpha3=-3、整体姿态角误差alpha4=-1.8、action change alpha5=-0.4、action magnitude alpha6=-0.3。  

必须注意：训练reward里没有直接计算电池电流、功率或焦耳。  
它用normalized PWM幅值作为能耗proxy，因此energy-aware更准确地理解为“鼓励少用和少突变推进器”，而不是根据精确电机效率图做最优能量控制。  

实验里的power也不是论文明确报告的电池端实测功率。  
作者根据Blue Robotics T200 datasheet，用四阶多项式把PWM映射成估计功率，再对8个推进器求和。  
因此“节能39%”是根据command推算的平均推进器功率降低，不应写成已经用电流传感器证明整机电能消耗减少39%。  

##### Stonefish仿真与训练数据从哪里来
仿真器是开源marine robotics simulator Stonefish，不是Isaac Lab。  
模型包括机器人几何、浮力、线性/二次阻力、推进器以及附加质量；附加质量通过近似ellipsoid计算。  
仿真和真实系统都通过LCM中间件传递状态与命令，以减少部署接口差异。  

训练数据不是预先采集的数据集，也没有使用真实机器人replay。  
TQC在线与Stonefish交互：policy给出8维action，仿真推进一步产生下一状态和reward，transition进入replay buffer，再采样更新actor/critics。  

训练条件：  
7个并行MOLA仿真实例  
7 million environment steps  
每episode 40s，共400 steps，因此控制/仿真步长为0.1s  
Intel 12代i7、64GB RAM、NVIDIA RTX 2000 Ada  
总训练时间约15小时  

每个episode初始pose在目标周围12m x 12m x 6m空间内随机，初始orientation不受约束。  
目标位置固定为(0,0,4m)，目标姿态从预先构造的36种orientation组合中采样。  
作者试过连续随机目标姿态，但学习效果更差，最后改成36种离散目标姿态；这说明policy并没有在训练中均匀覆盖整个SO(3)。  

##### Domain randomization实际随机了多少
REEF-DR只随机质量：nominal mass为30.6kg，每个episode在30.4kg到30.8kg之间均匀采样。  
这个范围约为nominal的正负0.65%，并通过质量变化同时改变净浮力和质心关系：30.4kg时偏正浮力，30.8kg时偏负浮力。  

它没有像更完整的sim-to-real pipeline那样随机化drag、added mass、current、thruster gain/dead-zone、sensor noise、latency或执行器故障。  
所以论文证明的是非常窄的mass / buoyancy不确定性鲁棒性，不能泛化为对任意payload和海洋环境变化都鲁棒。  

##### 仿真评估
作者用1152个30s episode系统评估控制器：  
初始x/y取-2m或2m，z取2m或6m  
初始roll/pitch/heading取0或90度  
目标位置为(0,0,4m)  
目标roll取0或30度、pitch取0 / 60 / 90度、heading取0 / 45 / -60度  
64种初始条件乘18种目标姿态，共1152组。  

比较对象包括手工调好的MOLA PID、已有SAC-based Sola controller、REEF和REEF-DR。  
nominal mass下的position / attitude RMSE大致为：  
PID：x 0.62m，y 0.62m，z 0.66m，orientation 14.32deg  
Sola：x 0.69m，y 0.69m，z 0.61m，orientation 98.55deg  
REEF：x 0.55m，y 0.55m，z 0.50m，orientation 18.33deg  
REEF-DR：x 0.59m，y 0.56m，z 0.53m，orientation 22.35deg  

因此不能笼统说RL所有指标都优于PID。  
REEF的transient position RMSE更低，但orientation RMSE比PID更高；REEF-DR在nominal dynamics下又略逊于固定模型REEF，这是robust policy学习多种动力学平均解的常见代价。  

质量改到30.8kg时steady-state position error大致为：  
PID 0.08m  
Sola 0.45m  
REEF 0.36m  
REEF-DR 0.14m  

这支持mass randomization在设定的质量范围内提高了鲁棒性，但30.8kg恰好仍位于REEF-DR训练区间边界，不是超出训练分布的强OOD测试。  

仿真估计平均功率：  
PID 228.8W  
Sola 1741.1W  
REEF 159.4W  
REEF-DR 151.9W  

REEF和REEF-DR相对PID分别低约30.3%和33.6%。Sola输出过于激进、估计功率异常高，因此作者出于安全没有把它部署到实物。  

##### 节能策略究竟学到了什么
轨迹图显示PID通常先迅速把机器人转到最终目标姿态，再沿该姿态平移。  
如果最终姿态让机器人以较大迎水面积侧移，这会产生更大阻力。  

RL controller学到的策略更像：  
先把艇体朝向有利于移动的方向  
主要利用前进/后退方向靠近目标  
接近目标后才完成最终姿态调整  

这解释了两个看似矛盾的结果：  
它用更小的推进器动作、更快到达位置，因此估计能耗较低  
它在运动途中没有立即保持最终目标姿态，因此整段trajectory上的orientation RMSE比PID更差  

所以节能不是“相同位姿轨迹下单纯提高电机效率”，而是policy改变了position与orientation两个目标的时间顺序，用暂时牺牲姿态跟踪换取更低阻力的移动方式。  

##### 真实水槽实验与sim-to-real
实物场地是MBARI约13m x 9m x 10m的测试水槽。  
机器人用IMU、DVL和depth sensing估计完整6-DoF位置与姿态，不依赖外部motion capture。  
作者从不同6-DoF运动中选择10组representative in-water episodes，比较PID、REEF和REEF-DR；Sola因仿真动作过激未下水。  

真实实验平均结果约为：  
PID：x 0.50±0.14m，y 0.58±0.03m，z 0.33±0.12m，orientation 9.17±2.86deg，settling time 30.28±7.53s  
REEF：x 0.46±0.13m，y 0.59±0.03m，z 0.33±0.09m，orientation 12.03±2.86deg，settling time 17.44±1.67s  
REEF-DR：x 0.45±0.14m，y 0.55±0.05m，z 0.36±0.15m，orientation 13.75±4.01deg，settling time 16.45±3.24s  

结论仍然是：RL的position accuracy与PID大致相当，settling更快，但orientation tracking较差。  

通过T200 PWM-power拟合得到的真实试验平均估计功率：  
PID 276.7W  
REEF 164.8W  
REEF-DR 168.1W  

对应约40.4%和39.2%的估计功率下降。  

论文描述的pipeline是仿真训练后直接部署，没有像Fan 2024 Improved TD3那样采集真实transition再混入replay buffer重训，也没有报告真实finetuning。  
因此按流程可以视为direct sim-to-real / zero-shot-like transfer。  
不过作者没有把zero-shot作为严格实验术语，也没有详细报告部署前是否手动调过state scaling、PWM限制或安全参数，写总结时最好用“未报告真实数据微调”而不是过度断言完全零调参。  

##### 与前后工作的关系
相对前期Toward论文：  
Swim4Real增加absolute attitude观测、36种目标姿态、更大的网络、mass DR版本、1152组仿真评估和10组实物水槽验证。  

相对Learning to Swim：  
两者都是6-DoF目标位姿调节，也都直接输出individual thrusters。  
Learning to Swim在Isaac Lab用PPO和2048并行环境训练约10-20分钟，输出CUREE的6个推进器并做zero-shot水池position hold。  
Swim4Real在Stonefish用TQC、7个并行环境训练约15小时，输出MOLA的8个PWM，重点研究动作平滑与能耗并进行更系统的PID比较。  
Swim4Real尝试复现/适配Learning to Swim方案但没有成功学到policy，作者将原因归于PPO、reward、episode和observation差异；这不能当成公平head-to-head，也不能据此证明TQC本质上优于PPO。  

相对Improved TD3 Path-Following：  
Improved TD3跟踪预定义path，LOS产生路径误差，policy输出4个motion-channel commands，并在实物前使用真实replay重训。  
Swim4Real没有path或LOS，只调节一个固定6-DoF goal pose，直接输出8个PWM，并且按论文所述不使用真实训练数据。  

相对Sim2Swim：  
Swim4Real控制position + orientation setpoint，direct-to-thruster，训练约15小时。  
Sim2Swim控制time-varying linear velocity + orientation，输出body wrench后再做thruster allocation，加入integral observation消除稳态误差，训练约3分钟。  
因此Sim2Swim更适合接在guidance / path planner下面执行连续路径，而Swim4Real更接近节能的6-DoF“到位并停住”控制器。  

##### 主要limitations
任务只是fixed setpoint regulation，标题中的agile不等于已验证任意时变trajectory tracking。  
没有障碍、感知、规划、unknown environment或open-ocean current。  
8维action及训练动力学绑定MOLA推进器布局，不能直接称platform independent。  
domain randomization只覆盖正负0.2kg质量，范围约正负0.65%，没有randomize主要水动力和传感执行器误差。  
能量reward使用PWM幅值proxy，39%来自datasheet拟合功率，而非明确的电池端实测能量。  
真实实验只有10组且是作者选择的representative episodes，没有大规模随机试验、成功率或置信区间。  
真实场景是受控水槽，不是有海流、波浪、通信和定位困难的open water。  
RL在orientation RMSE上差于PID，较低能耗部分来自延后最终姿态对齐，存在position / attitude / energy之间的任务偏好取舍。  
没有形式化稳定性或安全保证，已有SAC baseline甚至因动作过激而不敢下水。  
训练约15小时，速度显著慢于Learning to Swim和Sim2Swim的大规模GPU并行方案。  
截至整理时未找到作者公开的Swim4Real官方代码仓库。  

##### 对我们的启发
这篇最有价值的不是再证明一次RL可以把位置误差压到零，而是展示了6-DoF holonomic AUV可以通过联合优化位姿和推进器动作，自动学出与PID不同的energy-aware maneuver strategy。  

如果我们以后做水下navigation，可以借用它作为低层goal-pose controller，但需要在上层补齐：  
sonar / camera perception  
GPS-denied localization  
path / waypoint generation  
obstacle-aware guidance  
independent safety filter  

更进一步的研究问题是：不要只惩罚PWM，而是加入真实电流/电压、推进器效率曲线和任务完成时间，比较总能量Joule而非平均估计功率；同时让policy跟踪time-varying pose / velocity reference，并在真实海流和payload变化中测试。  

从我们当前的领域时间线看，Swim4Real应被放在Learning to Swim之后、Sim2Swim之前理解：  
Learning to Swim证明了快速并行RL的6-DoF direct-thruster zero-shot控制可行  
Swim4Real把重点转到能耗和动作策略，但训练更慢、任务仍是setpoint  
Sim2Swim再把接口推进到time-varying velocity + attitude tracking，并用wrench output提高跨平台通用性  

#### MarineGym: A High-Performance Reinforcement Learning Platform for Underwater Robotics
2025 Zhejiang / Heriot-Watt / Edinburgh / HKUST  
发表/收录：IROS 2025（IEEE/RSJ International Conference on Intelligent Robots and Systems） 论文页也标注获得 ICRA 2025 AQ2UASIM Workshop Best Paper Award  
这篇不是提出一个单独的水下控制policy 而是提出一个面向水下机器人强化学习的高性能仿真/训练平台  
作者想解决的是：现有水下仿真器要么不够RL-friendly 要么CPU并行效率低 要么缺少统一benchmark和domain randomization工具  
所以MarineGym更像水下版本的 Isaac Gym / Isaac Lab 风格平台：GPU并行仿真 + Gym接口 + PPO等RL训练 + UUV模型库 + benchmark任务  

#### 平台定位和方法能力
MarineGym的核心能力是让水下机器人RL可以大规模并行训练  
它基于NVIDIA Isaac Sim / PhysX 但Isaac Sim本身没有水动力 所以作者额外实现了GPU hydrodynamic plugin  
论文报告在单张RTX 3060上可以支持8000多个并行环境 rollout速度达到约250000 FPS  
这对水下RL很重要 因为传统Gazebo / DAVE / Stonefish / HoloOcean这类平台通常很难像腿式机器人那样一次并行几千个环境  

它解决的不是视觉导航/建图问题 而是“水下控制任务如何高效训练和评估”  
目前benchmark主要覆盖三类基础控制任务：  
1）station keeping：靠近并保持目标pose 抗水流/载荷扰动  
2）trajectory tracking：跟踪三维时间变化轨迹 比如helix / Lissajous曲线  
3）docking / landing：对准并落到水下平台上 同时保持接触稳定  

所以它和Learning to Swim / Fast Policy Learning更相关 是低层/中层控制训练平台  
它不是Nav2Goal那种真实图像输入到yaw/pitch动作的导航policy  

#### 仿真器 / 是否开源
MarineGym已经开源：GitHub仓库是 `Marine-RL/MarineGym` 许可证是MIT  
官方README显示它基于OmniDrones和Isaac Sim 训练脚本在`scripts/train.py`  
当前仓库README说已经验证的环境包括 Hover, Circle Tracking, Helical Tracking, Lemniscate Tracking, Landing  
也就是说论文里的station keeping / trajectory tracking / docking 在代码里大致对应Hover / Track / Landing这几类任务  
README还提到vision-based和sonar-based任务仍在开发中  
所以如果我们想复现水下RL控制 它是目前比很多零散水下仿真代码更值得优先看的平台  
但如果我们想做真实相机/声呐感知闭环 它现在还不算现成解决方案  

#### 水动力pipeline
Isaac Sim负责刚体动力学、碰撞和渲染  
MarineGym额外计算水动力项 并把力/力矩施加到每个submerged link的质心  
水动力模型基于Fossen equation of motion 也就是水下机器人常用的6-DoF动力学形式：  
惯性/附加质量 + Coriolis/centripetal + damping + restoring force + actuator force  

论文把动力学拆成两部分：  
Rigid-body term：由PhysX处理 包括刚体运动和碰撞响应  
Hydrodynamic term：由自定义GPU插件处理 包括added mass damping restoring forces等水动力效应  

水流扰动的处理方式也比较标准：  
先把海流速度投影到body-fixed frame 得到当前机器人相对水体的速度  
再用relative velocity替代原本速度进入水动力方程  
这样水流会直接改变阻力/水动力项  

payload扰动通过在机器人body上附加额外物体实现  
payload的质量和安装位置可以配置 这会改变整体质量分布和动力学  

#### UUV模型和执行器
平台提供五种UUV模型 覆盖三类典型结构：  
BlueROV和BlueROV Heavy：multirotor / 多推进器ROV类型 适合精细操作和悬停  
LAUV和iAUV：rudder-propeller / 主推进器+舵面类型 更偏高速巡航 但欠驱动  
HAUV：tiltrotor hybrid aerial underwater vehicle / 倾转旋翼空水两栖机器人  

执行器模型不是简单把动作直接当力  
它把执行器拆成两个模块：  
rotor dynamics model：输入命令到转速响应 可以是zero-order first-order 或实验数据训练的NN模型  
thrust generation model：由转速/舵角生成实际推力和力矩  

推进器推力模型包含二次函数和dead zone  
舵面模型用简化lift / drag formulation  
仓库/论文还给了Blue Robotics常用T200和2820推进器的实验参数  

这点对复现有意义：  
如果我们直接让policy输出body wrench 或者理想thruster force 训练会更容易  
但真实部署时还需要考虑推进器响应、死区、安装位置和分配矩阵  
MarineGym试图把这些执行器层面的细节也放进仿真配置里  

#### Domain randomization
MarineGym的DR toolkit比较完整 分成四类参数：  
physical properties：质量、惯量、质心等  
simulation settings：流体密度、added mass matrix、damping matrix等  
actuator parameters：time constant、force constant、安装位置等  
external environment：水流速度/方向、外部载荷质量和位置等  

采样分布支持uniform Gaussian 和自定义piecewise functions  
DR范围还可以在训练过程中动态调整 因此可以做curriculum learning  
这一点比前面一些水下控制论文只randomize浮力/重力要完整很多  

论文里的benchmark设置有三档：  
standard environment  
disturbance environment：加入水流和payload变化  
disturbance + randomization：在扰动基础上进一步做domain randomization  

#### 观测输入和输出
benchmark任务被建模为MDP 并通过Gym-style interface给RL算法使用  
作者说state space只包含UUV自身kinematic information以及必要task-related parameters  
也就是当前位姿/速度/目标位姿或轨迹参考等结构化状态  
不是RGB图像 也不是声呐图像  

action space是所有执行器的control commands  
所以策略学的是 state observation -> actuator commands  
不是先输出高层heading 再由手写控制器执行  
也不是像Fast Policy Learning那样只输出body wrench再分配到推进器  

这意味着MarineGym里的baseline更偏端到端控制  
但这里的“端到端”是状态到执行器 不包含视觉/声呐感知端到端  

#### Benchmark实验
速度对比：  
BlueROV2 Heavy station-keeping任务上 MarineGym在RTX 3060上达到10^5量级FPS  
论文表格中约2000环境时约1.7e5 FPS 约4100环境时约2.1e5 FPS 约8200环境时约2.5e5 FPS  
相比之下 DAVE / Stonefish / HoloOcean通常是10到100 FPS量级 并行DAVE大约10^3 FPS量级  

五种UUV x 三个任务 x 三种环境难度都做了训练曲线  
总体结论是所有模型最终都能收敛  
但LAUV和iAUV这种欠驱动rudder-propeller模型大概要多约两倍训练步数  
加入扰动和DR后 大多数场景性能会下降  
BlueROV Heavy和HAUV因为8执行器/机动性强 性能下降最小  
LAUV和iAUV在强水流下更难保持位置和稳定  

DR评估是Sim2Sim：  
同一任务训练DR policy和NDR policy  
测试到Env1（在DR范围内）和Env2（超出DR范围）  
指标是station keeping position error / trajectory tracking mean deviation / docking final contact error  
结果是DR policy在三类任务里都更鲁棒  
比如docking任务在Env2里 DR相对NDR最多降低约64%误差  

#### limitation / future work
最大limitation：没有真实机器人验证  
论文证明的是高性能仿真训练和Sim2Sim鲁棒性 还没有证明Sim2Real到真实水池/真实海洋  

第二个limitation：任务类型仍然基础  
目前主要是hover / trajectory tracking / docking 还不是复杂长期导航、动态避障、真实任务级自主系统  

第三个limitation：水下视觉特性还不完整  
虽然Isaac Sim可以做ray tracing underwater visual effects 但作者明确说关键水下视觉特性如light propagation和turbidity还没有完全复现  
这对我们很重要 因为如果要做camera / sonar based navigation 仅有动力学平台还不够  

future work：  
Sim2Real validation  
更复杂、更真实的水下场景  
扩展到更困难的任务  
仓库README也提到vision-based和sonar-based任务仍在开发中  

#### 对我们的启发
MarineGym对我们最有价值的是“平台/benchmark” 而不是一个新policy结构  
如果我们想做水下learning-based motion control 它可以作为复现实验和新方法验证的起点  
尤其适合做：6-DoF控制、trajectory tracking、docking、payload/water-current鲁棒性、不同UUV结构泛化  

但如果我们的目标是“相机/声呐感知 + learning navigation” 它目前还只是底座  
我们可能需要在MarineGym上补一个teacher-student pipeline：  
teacher用privileged state / target / obstacle geometry训练高性能控制或导航  
student用camera / sonar / depth / range观测蒸馏teacher  
这条路会比直接从真实传感器端到端RL现实很多  

和前面几篇的关系可以这样看：  
Nav2Goal：真实图像 + relative goal -> 离散yaw/pitch 是早期视觉反应式导航  
Hadi TD3 navigation：理想化障碍物距离 + goal/state -> continuous rudder 是6-DoF动力学上的2D仿真mapless navigation  
Learning to Swim：Isaac并行水下控制器 状态 -> thruster 真机zero-shot  
Sim2Swim：Isaac并行训练 目标3D速度+姿态 -> body wrench 真机zero-shot敏捷path following  
Fast Policy Learning：MJX快速训练 状态误差 -> body wrench 真机动捕反馈  
MarineGym：更系统的平台化水下RL benchmark 目标是把水下控制训练规模化、标准化  

链接：https://marine-gym.com/  
代码：https://github.com/Marine-RL/MarineGym  

#### Deep Reinforcement Learning for Autonomous Underwater Navigation: A Comparative Study with DWA and Digital Twin Validation
2025/2026 BlueROV2  
预印本题名：Digital Twin–Supervised Reinforcement Learning Framework for Autonomous Underwater Navigation  
作者：Zamirddine Mari, Mohamad Motasem Nawaf, Pierre Drap  
单位：DGA Techniques Navales；LIS, CNRS / Aix-Marseille University  
正式发表：Sensors 2026, 26(7), 2179，2026-04-01发表，DOI 10.3390/s26072179  
时间线上按2025-12首次公开的预印本放置，但引用时应使用2026 Sensors正式版本；旧总结中“尚未确认接收”已经过期  

期刊定位：Sensors是MDPI旗下覆盖面很广的传感器综合期刊。按2026年公布的2025 JCR数据，Impact Factor 4.0，在Instruments & Instrumentation等类别为JCR Q2；Scopus CiteScore指标相对更高。它是正规SCI/SCIE期刊，但在机器人与控制领域的社区认可度明显低于IEEE TRO、RA-L、JOE以及ICRA/IROS等主要机器人期刊和会议。该刊发文量大、审稿与出版较快，不同论文质量差异也较大。因此不能因为期刊有Q1/Q2指标就把这篇视作水下导航state of the art，仍应根据任务完整度和实验链条判断。

一句话结论：这是一篇**固定深度的二维高层导航/虚拟避障**论文。PPO读取目标几何信息、数字孪生生成的虚拟障碍栅格和工作区边界距离，输出7档离散航向增量；训练转移模型是二维定步长运动学。海试确实让真实BlueROV2执行了RL航向决策，但依赖USBL、缆绳、岸上计算与虚拟障碍，并未验证真实声呐/相机障碍感知。

#### 这篇到底在做什么任务
场景是一个矩形或四边形工作区：  
机器人从入口一侧出发，在固定深度向另一侧的出口gate移动  
工作区内随机放置大约5到10个静态障碍物  
成功条件是穿过目标gate；失败条件是撞障碍或驶出工作区  

学习问题只保留水平面状态：

```text
s(t) = [x(t), y(t), theta(t)]
```

其中theta是heading。深度固定，roll和pitch不进入RL导航问题。  
所以它不是6-DoF motion control，不是任意三维goal reaching，也不是预给定轨迹tracking；更准确地说是**二维反应式local navigation / gate reaching**。

论文把环境称作partially unknown，含义是每个episode的障碍布局会变化，policy不是预先记住一张固定地图。  
但在每个时刻，障碍物和边界信息由虚拟环境根据已知几何直接生成给policy；因此这并不是机器人在真实未知水下环境中自行感知和建图。

#### 整体系统分层

```text
USBL位置(x,y) + IMU航向/深度 + Unity数字孪生中的虚拟障碍与边界
                              ↓
             归一化并拼成84维结构化观测
                              ↓
                 PPO选择7档航向增量之一
                              ↓
                   运动学/边界安全检查
                              ↓
                      MAVLink命令
                              ↓
       BlueOS / ArduSub负责低层姿态稳定、控制与推进器执行
```

RL承担的是高层“下一步转多少”的决策。  
ArduSub等传统模块承担从航向命令到真实推进器动作的低层闭环，因此论文不是重新学习BlueROV2的thruster allocation或6-DoF动力学控制。

标题中的supervised不是监督学习supervised learning。  
这里是“由数字孪生监督/监视真实实验”的意思：数字孪生提供虚拟环境信息、实时可视化和安全实验边界，而不是用带标签数据监督PPO。

#### ArduSub、USBL和MAVLink分别是什么

| 组件 | 在系统中的作用 | 是否为学习方法 |
|---|---|---|
| USBL | 从水面通过声学测距和测角估计BlueROV2水下位置 | 否 |
| Unity数字孪生 | 根据USBL位置计算虚拟障碍物和工作区边界信息 | 否 |
| PPO | 根据目标与虚拟几何选择下一步离散航向变化 | 是 |
| MAVLink | 在岸上计算机与BlueROV2之间封装和传输状态、参数与控制命令 | 否 |
| ArduSub | 姿态/航向稳定、定深、推进器混控、执行航向命令和failsafe | 否，主要是传统控制 |

ArduSub是ArduPilot面向ROV的开源autopilot firmware。它可以运行在Pixhawk等飞控上，使用IMU、压力传感器和可选外部定位完成姿态、航向、深度甚至位置闭环，并把forward/lateral/heave/roll/pitch/yaw等命令分配给不同推进器。BlueROV2 Heavy的8推进器6-DoF布局已经受到ArduSub支持。它内部主要使用经过工程调参的传统级联控制器/PID，而不是这篇论文训练的PPO。

USBL是Ultra-Short Baseline超短基线声学定位。水面换能器向机器人上的声学应答器发ping，往返时间给出range，换能器阵列的到达时间/相位差给出bearing；再结合水面端GPS、heading和姿态，就能估计水下机器人的全局位置。它解决了水下没有GPS的问题，但属于依赖水面基站的外部定位基础设施，不等于机器人仅靠机载传感器自主定位；精度还会受声速、多径反射、安装误差和水面运动影响。

MAVLink只是轻量级二进制通信协议，可以传输position、attitude、telemetry、parameter、mission和control command。它不是控制算法，也不是通信物理介质；本实验中消息实际通过tether链路传输。可以把MAVLink理解为“消息语言”，tether/Ethernet是“道路”，ArduSub才是收到命令后控制推进器的“驾驶员”。

#### ArduSub已经能控制机器人，为什么还需要RL
这个问题需要区分guidance/navigation与low-level control：  
ArduSub已经可以可靠完成常规姿态稳定、定深、位置保持和航点执行；这篇PPO没有取代这些能力，只是在它上面增加“面对目标和障碍，下一步向哪里转”的高层local planner。因此这篇应当与DWA、人工势场、MPC local planner等导航算法比较，而不是声称PPO比ArduSub控制得更好。

Learning to Swim、Sim2Swim、Swim4Real等低层RL论文研究的是另一个问题：希望在payload、水动力参数和环境扰动变化时，减少传统PID重新调参，提高6-DoF敏捷性、稳态精度或能效。但对普通ROV任务，一个调好的ArduSub/PID往往已经足够，而且更容易验证和保证安全。RL只有在和强传统基线对比后明确证明更准、更鲁棒、更节能或更少调参，才有替代低层控制器的必要性。

这也进一步削弱了本文PPO部分的水下特殊性：水动力和推进器执行已经交给ArduSub，PPO只看到二维位置、目标和虚拟障碍几何。

#### 仿真器和动力学：最容易被标题误导的部分
预印本Figure 1明确把训练/对比环境称为2D Python visualization；正式版后文又使用realistic 3D environment等表述。  
但论文真正写出的RL状态转移是二维离散运动学：

```text
theta(t+1) = theta(t) + Delta-theta_a
x(t+1) = x(t) + Delta-d cos(theta(t+1))
y(t+1) = y(t) + Delta-d sin(theta(t+1))
```

机器人每一步以固定速度/固定步长向前，只改变heading。  
模型中没有质量、惯量、浮力、附加质量、Coriolis项、水阻、水流、推进器动力学、执行器延迟或传感器噪声。  

因此要把两种“环境”分开：

1. **RL训练/算法对比环境**：核心转移是二维运动学，用来快速生成轨迹和碰撞结果。  
2. **海试数字孪生**：真实场地的photogrammetry三维模型导入Unity，用于同步机器人avatar、生成虚拟障碍/边界信息和监督实验。  

三维photogrammetric场景精细，不等于RL训练具有高保真水下动力学。论文最后关于“在3D环境训练”的表述与其明示的二维运动学模型并不完全一致；按可复现的数学定义，应把它归类为**二维运动学导航训练 + 三维数字孪生海试监督**。

#### Observation：84维到底是什么
观测由三部分拼接：

1. **目标信息2维**  
   归一化目标距离，以及机器人heading相对目标方向的角度误差。  
   值得注意的是，公式把角度误差取绝对值再除以pi；若代码确实如此，goal cue会丢掉目标在左侧还是右侧的符号，方向只能借助其他结构或策略历史消歧。这可能是公式或实现描述不完整。

2. **虚拟极坐标occupancy grid**  
   按距离bin和角度bin，把相应扇区是否存在障碍编码为0/1。论文说它“inspired by forward-looking sonar”，但它不是FLS原始图像、声呐点云，也没有多径、散斑、漏检、时延等声学效应。正式版也明确把真实sonar point cloud列为未来工作。

3. **边界raycasting**  
   从机器人向多个方向发射虚拟射线，计算与四边形工作区边界的首次交点距离并归一化，让policy知道哪里会出界。

这套输入属于privileged geometric observation：数字环境已经替机器人解决了“哪里有障碍、边界在哪里”的感知问题。  
论文给出了总维度84以及`2 + n + q = 84`，但没有明确报告occupancy grid的扁平维数n和边界射线数q；图中的距离/速度/角度索引与公式也没有完全对应，因此84维的精确拆分仍不够可复现。

#### Action：网络输出什么
PPO动作是7个离散heading变化：

```text
{-45°, -30°, -15°, 0°, +15°, +30°, +45°}
```

选择后更新heading，再按固定步长前进。  
所以它不输出电机PWM、不输出单个推进器转速、不输出6维body wrench，也不连续调节线速度。  
它和Nav2Goal的高层离散转向更接近，而不是Learning to Swim / Sim2Swim的低层控制器。

#### Reward
Reward由三层组成：

1. local progress：只有向出口取得正向进展时给正奖励；如果不前进或后退，reward为0而不是负值。  
2. milestones：越过25%、50%、75%进度时分别给一次额外奖励。  
3. terminal reward：到达出口gate给成功奖励；撞障碍或出界给失败惩罚并终止。

作者说明成功奖励被设为显著大于累计进展奖励，碰撞惩罚也足以压过冒险抄近路的收益，但没有公布各reward系数的具体数值。  
Reward没有直接惩罚时间、路径长度、动作变化、能耗、推进器负担或水流影响，因此它学习的是“前进、别撞、别出界”，不是能量最优或动力学平滑控制。

#### PPO和训练设置
实现使用Ray 2.49.2 / RLlib PPO。  
Policy与value network是MLP，隐藏层为[128, 128, 128]，tanh激活，观测归一化，actor和critic使用独立head，没有CNN。  

正式版给出的主要参数：learning rate 2.5e-4，gamma 0.95，GAE lambda 0.9，clip 0.3，KL target 0.01，entropy coefficient 0，30 epochs，rollout fragment 30，train batch 1950，minibatch 128。  
预印本还报告训练电脑为Windows 11、Intel i9-10900K、RTX 3080、32 GB RAM。  

训练约7000/7020 iterations。论文没有报告wall-clock训练时间、总environment steps、episode最大步数、随机种子数量或多次独立训练的均值/方差。  
正文没有公开代码仓库；正式版Data Availability写的是dataset available on request，所以目前不能标为开源。

#### DWA对比是否公平
论文把DWA作为传统基线。它对候选`(Delta-theta, d)`做一步运动学预测，再综合：

- 到目标的距离；
- 与最近障碍物的clearance；
- 向前行进距离。

然后选择得分最高的候选动作。  
这个实现比经典DWA中根据当前速度、加速度约束构造dynamic window并滚动预测轨迹的完整版本更简化，更像**离散一步kinematic local planner**。论文也没有给出DWA权重alpha、beta、gamma及调参过程。

因此PPO确实击败了“论文实现的DWA”，但不能据此推出PPO普遍优于成熟DWA/TEB/全局-局部规划系统。一个成功率只有8%的基线也提示DWA配置、障碍密度或局部极小值问题很严重。

#### 仿真结果
训练阶段success rate统计：mean 73.4%，median 74.5%，maximum 79.6%。  
这里是训练过程中不断变化的policy在随机episode上的统计，不是固定policy的独立held-out测试成功率。

固定100个较难场景中，两种方法使用相同障碍布局：

| 方法 | 成功 | 碰撞 | 出界 |
|---|---:|---:|---:|
| DWA | 8% | 76% | 16% |
| PPO | 55% | 17% | 28% |

PPO显著减少碰撞并提高成功率，但失败仍有45%，且出界率比DWA更高。  
正式版解释训练与测试的差距：训练平均5–10个障碍、密度约0.003 obs/m²；测试固定10个、密度约0.005 obs/m²，被作者称为高约40%的stress test。  

需要保留三个统计局限：只有100个测试episode；没有多随机种子/置信区间/显著性检验；没有报告真实数值的路径效率和安全距离，Table 4只是列出这些指标的意义。

#### 海试到底验证了什么
海试地点：法国Marseille Pointe Rouge港。  
平台：BlueROV2 Heavy，带IMU、压力深度计、前视相机和SeaTrac USBL。  
USBL提供亚米级`(x,y)`位置；机器人通过中性浮力tether连接水面控制站，传输实时telemetry和RL命令。  
RL模块运行在水面侧系统，通过MAVLink把航向调整交给BlueOS / ArduSub执行。

最关键的实验限制是：**海试障碍物全部只存在于数字孪生里，没有在水中放置物理障碍物。**  
论文明确写明obstacles were rendered exclusively within the 3D digital twin。  
前视相机用于视频/视觉对照，不是policy输入；virtual occupancy grid和ray distances由数字孪生提供，不是实时声呐或相机检测结果。

因此海试真正验证的是：

- USBL能把真机位置同步进数字孪生；
- PPO能根据目标和虚拟障碍信息输出航向决策；
- MAVLink / ArduSub能让真机大体执行这些命令；
- 在港口水动力和USBL噪声下，真实轨迹与规划轨迹的MAE约0.85 m。

它没有验证：真实障碍检测、真实物体碰撞安全、无USBL定位、无缆自主运行、动态障碍、三维避障或端到端机载感知。

#### 数字孪生是怎么建立的
正式版补充了较完整的场地建模数据：  
潜水员使用Nikon D800和14 mm镜头采集约26,000张图像；Agisoft Metashape完成photogrammetry；覆盖约2967 m²，地面分辨率0.313 mm/pixel；使用6个尺度尺，重建RMS为3.2 mm；最终mesh超过400万面并导入Unity。

这些数字说明三维场地模型本身很精细，但不能把3.2 mm的重建误差理解成机器人定位或控制精度。  
真机闭环的位置仍来自亚米级USBL，最终轨迹MAE是0.85 m；两者属于不同误差层级。

数字孪生在这篇中的主要价值是：

- 提供已知场地几何、虚拟障碍和边界射线；
- 同步显示真实BlueROV2 avatar和虚拟相机；
- 在不放置真实障碍的条件下降低海试风险；
- 对比planned trajectory与USBL tracked trajectory。

它不是用来生成真实水动力数据的CFD simulator，也没有完成在线system identification。

#### 论文声称与实际证据的边界
论文能支持的结论：  
在作者定义的二维运动学虚拟导航benchmark里，PPO比其简化DWA基线更能绕过密集静态障碍；这套高层策略可以通过USBL + 数字孪生 + MAVLink / ArduSub链路驱动真实BlueROV2，轨迹误差约0.85 m。

论文不能充分支持的更强结论：  
“真实未知水下环境自主避障”“真实声呐/视觉感知”“高保真3D sim-to-real导航”“无需外部定位”“完整AUV autonomy”。作者实际上也把真实video/sonar、3D/6-DoF、safe RL、multi-agent和visual relocalization全部列为未来工作。

所以这篇应标为：**有真机HIL/海试，但真实验证的障碍是虚拟的；不是纯仿真论文，也不是完整真实避障论文。**

#### 放在水里和放在陆地上有多大区别
对PPO算法核心而言，区别很小。只要把BlueROV2替换成一辆具有二维位置、heading、固定前进步长和低层航向控制器的地面小车，84维结构化观测和7档离散转向policy基本不需要改变。它没有在学习问题中使用浮力、阻力、附加质量、水流、推进器状态或三维运动等水下特有变量。

这篇真正的水下内容主要位于外围工程系统：水下无GPS所以使用USBL；无线电传播困难所以使用tether；真实平台由ArduSub稳定；测试场地通过水下photogrammetry建立数字孪生；实际轨迹受到水流和声学定位噪声影响。但这些因素大多没有进入PPO训练模型，也不是学习算法解决的对象。

因此更准确的学术定位是：**将通用二维RL局部导航器集成进USBL—数字孪生—MAVLink—ArduSub—BlueROV2链路，并完成虚拟障碍条件下的水下HIL/真机执行验证。**它的贡献更偏工程集成与安全验证，不是新的水下动力学控制方法，也不能证明完整真实水下自主避障。

#### 写作和复现上的问题

1. 84维只给总维数，没有把occupancy维数n和boundary-ray维数q明确展开。  
2. 目标角误差公式使用绝对值，可能丢失左右符号；论文未解释。  
3. occupancy描述先引入速度、距离、角度三组离散量，但最终indicator只有距离和角度索引。  
4. 训练动力学是二维定步长模型，结论却多次称3D realistic environment，容易把三维场景渲染与三维动力学混为一谈。  
5. DWA权重与调参过程、reward具体系数、真实控制频率、安全检查如何调整动作均未完整报告。  
6. 海试没有报告实验次数、成功率/碰撞率等完整统计，只给planned-vs-USBL path MAE 0.85 m和定性截图。  
7. 预印本和正式版部分超参数/Unity版本表述有变化，复现应以正式版为准，但正式版仍未开放代码。

#### 和前面工作的关系
Nav2Goal：真实前视图像 + relative goal -> 离散yaw/pitch；真实珊瑚礁障碍感知更接近完整导航，但仍依赖相对定位和恒速运动。  
Hadi 2022：理想障碍距离 + goal/state -> 连续rudder；只有仿真，但至少使用REMUS 6-DoF动力学执行二维定深任务。  
这篇Digital Twin：结构化虚拟几何 + goal -> 离散heading；加入了真实BlueROV2执行和数字孪生HIL，但动力学训练更简化、真实障碍感知仍未做。  
Learning to Swim / Sim2Swim / Fast Policy Learning：解决的是低层pose/velocity control，不负责障碍感知与导航决策，可作为这篇高层policy下面的控制器。

#### 对我们的启发
这篇真正有价值的不是PPO网络结构，而是它暴露了水下导航中最难跨越的一段：

```text
仿真/数字孪生中的精确障碍几何
                ↓ reality gap
真实声呐/相机的噪声、漏检、多径、低能见度和定位漂移
```

一个更有研究价值的后续方向是teacher-student：  
训练时teacher使用数字孪生的privileged occupancy、边界和准确pose；student只使用FLS / imaging sonar、camera、IMU、DVL/depth等机载观测，通过蒸馏或asymmetric actor-critic学习；最后再用真实传感器噪声、时延、漏检和水动力domain randomization做sim-to-real。  
这会直接补上本文尚未解决的“真实感知到导航动作”链路。

正式论文：https://doi.org/10.3390/s26072179  
预印本：https://arxiv.org/abs/2512.10925

#### Fast Policy Learning for 6-DOF Position Control of Underwater Vehicles
2025/2026  
发表/收录：arXiv preprint 暂未看到正式会议/期刊收录信息  
这篇也是6-DoF position control 用JAX + MuJoCo-XLA / MJX做GPU加速RL训练 目标是两分钟内训练出水下6-DoF轨迹跟踪/扰动抑制policy 并做真实水下实验zero-shot transfer  
对我们关心的运动控制很相关 可以作为Learning to Swim后续同类工作看  
论文没有给GitHub/code链接 我查了一下目前也只看到arXiv页面 所以这篇的训练pipeline/仿真环境应该不能确认开源  
注意这里的“仿真器”不是作者从零写一个完整水下机器人仿真器 而是基于JAX + MJX（MuJoCo-XLA MuJoCo的JAX版本）搭建一个GPU并行AUV控制训练环境  

#### 任务和方法能力
任务是6-DOF position control：给定目标位置和姿态 policy输出控制量 让AUV同时稳定平移和旋转误差  
它不做视觉导航 不做目标检测 不做路径规划 不处理障碍物  
能力验证包括两个真实实验：  
1）Center Locked Helix Tracking：让BlueROV2 Heavy沿螺旋轨迹运动 同时机体朝向螺旋中心  
2）Disturbance Rejection：让机器人保持固定6-DoF pose 人为拉动tether制造外部扰动 看能不能回到setpoint  
所以这篇能力更偏“高性能水下低层/中层position controller” 而不是完整自主任务系统  

#### 仿真器 / 训练框架
训练框架基于JAX + MJX  
MJX是MuJoCo的JAX-compatible版本 可以在GPU上并行跑大量物理环境  
作者把AUV建成一个free-floating body 用ellipsoid geometry 表示 在MJX里有full 6-DOF dynamics  
通过JAX vectorization同时跑多个AUV环境 最高实验到4096 parallel environments  
同时用JIT compilation把physics rollout和learning updates一起编译 降低Python overhead  
在RTX 4060上 PPO在足够并行环境下能在几分钟内收敛 论文摘要说under two minutes 具体实验里多数大并行PPO在3分钟内达到稳定控制  

水动力建模：  
用了MJX内置的inertia fluid drag model 流体密度和粘度设置成水  
每个body通过mass/inertia tensor转换成等效inertia box 然后计算quadratic drag和viscous resistance  
浮力没有显式建模 而是通过修改effective gravity近似 并假设center of gravity和center of buoyancy对齐  
这点是它后面真实pitch误差的主要来源 因为真实机器人重心/浮心轻微不对齐会产生pitch restoring moment  

#### 输入输出
这篇和Learning to Swim有一个关键差别：它不是直接输出每个thruster command  
policy输出body-fixed frame下的6维wrench：  
A = {Fx, Fy, Fz, tau_roll, tau_pitch, tau_yaw}  
也就是surge/sway/heave三个方向的力 + roll/pitch/yaw三个力矩  
每个action归一化到[-1,1] 运行时scale到最大允许force/torque  
然后真实部署时再通过thruster allocation matrix把body wrench分配到各个推进器 最后转PWM  
这样比direct-to-thruster更通用 因为policy不绑定某个具体thruster布局 但代价是需要额外的allocation model  

观测是12维error-based observation：  
position error in body frame: ∆x, ∆y, ∆z  
attitude error: reference和当前orientation之间的3D axis-angle vector  
linear velocity in body frame: u, v, w  
angular velocity in body frame: p, q, r  
作者强调obs和action都在body-fixed frame里 这样policy不用学world/body坐标变换 学习更稳定  
他们没有做normalization 因为error-based formulation本身尺度比较接近  

#### RL算法比较
作者比较三类算法：  
PPO（Proximal Policy Optimization on-policy）  
SAC-DroQ（Soft Actor-Critic with Dropout Q-functions off-policy）  
SHAC（Short Horizon Actor-Critic differentiable-simulation based）  
选择这三类是为了比较on-policy / off-policy / differentiable simulation三种范式在MJX并行水下控制里的表现  
结果：三者都能学到6-DOF控制 但SHAC在真实实验里RMSE最低 PPO和DroQ也表现强 MPC baseline最差  
PPO受并行环境数量影响很大 大并行下很快收敛  
SHAC因为要反传物理模型 单次训练更慢 但样本效率高 512环境附近较合适  
DroQ对并行扩展收益没PPO那么直接 小环境数不收敛 大并行后能收敛但更慢  

#### Reward
reward目标是准确6-DOF控制 同时避免激进动作  
总reward：r = r_pos + r_att + r_act + r_vel + r_act-mavg  
r_pos：惩罚body-frame position error平方 可以对surge/sway/heave不同权重  
r_att：惩罚axis-angle attitude error平方  
r_act：惩罚torque command norm 约束控制量  
r_vel：惩罚angular velocity 避免高频旋转/振荡  
r_act-mavg：惩罚当前action偏离近期action moving average  
最后这个smooth action项对sim-to-real很重要 防止policy学到仿真可行但真实硬件危险的bang-bang高频控制  

#### Domain randomization
这篇domain randomization非常简单：随机化gravitational acceleration 来近似浮力变化  
前提是假设center of buoyancy和center of gravity对齐  
这只能模拟effective buoyant force变化 不能模拟浮心/重心错位产生的力矩  
所以它比Learning to Swim里randomize volume和CoB-CoM offset更简单 也解释了真实pitch上出现steady-state offset  

#### 真实实验
平台是BlueROV2 Heavy  
真实实验在大学室内水池做 用Qualisys underwater optical motion tracking提供ground truth feedback 这个feedback同时用于控制和评估  
所以这篇真实部署也不是完全onboard autonomous sensing 它依赖外部水下动捕来给pose反馈  
控制集成用ROS2 policy/MPC输出body-frame force/torque 然后通过allocation matrix转成individual thruster command 再插值成PWM  

Center Locked Helix Tracking：  
车辆沿helix trajectory运动 每0.25s给一个waypoint 同时身体持续朝向helix center  
这个任务会同时激发平移DOF和yaw 并通过耦合间接影响roll/pitch  
SHAC最好：3D position RMSE 0.099m attitude RMSE 9.79deg  
PPO 0.164m / 12.05deg DroQ 0.157m / 11.16deg MPC 0.315m / 17.51deg  

Disturbance Rejection：  
保持固定6-DOF pose 人工拉tether制造冲击扰动  
SHAC同样最好：3D RMSE 0.084m attitude RMSE 8.54deg  
PPO 0.150m / 10.00deg DroQ 0.192m / 9.70deg MPC 0.232m / 22.20deg  

#### limitation / future work
真实实验中pitch轴有明显steady-state error 大pitch command时pitch会残留offset  
作者认为原因是sim-to-real gap：仿真里假设center of gravity和center of buoyancy对齐 真实中轻微不对齐会带来pitch restoring moment  
future work是把offset buoyancy effects加入仿真器 验证这个假设  
它的真实控制依赖Qualisys外部动捕 不代表仅靠机载传感器闭环  
动作输出是body wrench 不是thruster-level policy 所以需要thruster allocation和motor dynamics在部署端处理  
reward没有为每个算法精细调参 作者也承认不同算法对reward敏感 当前比较不是各方法性能上限  
任务仍然是position control/trajectory tracking/抗扰 不是完整水下导航任务  

#### 和Learning to Swim对比
Learning to Swim：Isaac Lab自建水下仿真器 直接输出6个thruster commands 真实用AprilTag+DVL+IMU EKF 状态估计 代码明确开源  
Fast Policy Learning：JAX+MJX并行训练 输出body-frame force/torque 再通过allocation matrix到thruster 真实用Qualisys动捕反馈 论文未看到代码开源  
Learning to Swim更像direct-to-thruster controller验证 Fast Policy Learning更像“通用body wrench controller + 超快JAX/MJX训练框架”  
链接：https://arxiv.org/abs/2512.13359

#### Sim2Swim: Zero-Shot Velocity Control for Agile AUV Maneuvering in 3 Minutes
2025/2026 SINTEF Ocean  
发表/收录：已被IFAC World Congress 2026接收 安排在Next-Generation Control and Autonomy for Marine Systems and Vehicles session；截至2026-08-25会议正在进行 正式IFAC-PapersOnLine论文集/DOI尚待更新  
arXiv v1是2025-12-09 v2是2026-06-29  
这篇明确建立在Learning to Swim之上 但把任务从6-DoF position/pose control改成了3D linear velocity + orientation control  
核心目标是让body-frame线速度误差趋近0 同时让姿态四元数误差趋近identity quaternion  
它仍然是低层/中层运动控制器 不处理图像/声呐 不做避障或路径规划  

#### 任务和方法能力
目标机器人是fully actuated holonomic AUV 可以在surge/sway/heave三个平移方向独立运动 同时控制roll/pitch/yaw姿态  
controller接收随时间变化的目标body velocity v_d^b(t)和目标orientation q_d(t) 让实际速度和姿态跟踪这些reference  
形式化目标是：  
linear velocity error v_e^b = v^b - v_d^b -> 0  
quaternion error q_e = conjugate(q_d) * q -> identity quaternion  

这篇本身不是position controller 因为policy没有直接接收目标位置  
真实实验中的path following来自policy外部的3D LOS（line-of-sight）guidance：  
desired path -> LOS guidance -> desired body linear velocity -> Sim2Swim -> body wrench -> thruster allocation  
目标姿态可以由LOS给出 也可以独立指定 因此机器人运动方向和机体朝向可以解耦  
这种能力适合近距离结构检查：AUV沿一条路径移动时 可以侧身/俯仰去观察不与路径平行的表面  

#### 仿真器 / 训练框架
训练环境基于NVIDIA Isaac Lab 使用GPU massively parallel simulation  
算法是RSL-RL实现的PPO policy是2-layer MLP  
一次并行2048个AUV环境 每个episode最多5秒  
训练硬件：Intel i7-12800HX + NVIDIA A2000 8GB VRAM + 32GB RAM  
policy约80秒达到收敛 整个训练在3分钟内完成 最后一次iteration的mean reward约315  
相比Learning to Swim在A6000上训练约10-20分钟 Sim2Swim用更弱的笔记本级GPU获得接近一个数量级的训练加速  

训练时目标速度大小固定为0.5m/s 方向在3D unit sphere上随机采样：  
||v_d^b|| = 0.5m/s  
也就是速度大小固定 但可以随机向前后/左右/上下或任意组合方向运动  
目标姿态是time-varying reference 带随机初始条件 并沿轨迹的Frenet-Serret frame变化  
用于生成轨迹的速度形式是v(t) = [a, b sin(omega t), c cos(omega t)] 参数[a,b,c]=[0.5,0.5,0.3] omega=0.2  

论文没有像Learning to Swim那样给出明确的官方代码仓库 目前只确认arXiv论文和演示信息  
截至2026-08-25应标记为：训练pipeline / Isaac Lab环境未确认开源  

#### 观测和积分增强
policy observation是16维：  
o = [q_e, v_e^b, omega^b, z_v, z_q]  
q_e：目标姿态和当前姿态的quaternion error 4维  
v_e^b：body-frame linear velocity error 3维  
omega^b：当前angular velocity 3维  
z_v：linear velocity error的积分状态 3维  
z_q：quaternion error vector part的积分状态 3维  

z_v和z_q是这篇相对Learning to Swim最重要的改动  
普通MLP没有时间记忆 只看瞬时误差时 无法区分短暂扰动和长期固定偏差  
加入积分状态以后 持续的小误差会不断累积 policy可以据此增加持续的force/torque 去抵消固定浮力、载荷、水阻或恢复力矩  
当瞬时误差回到0时 积分状态仍可保留非零值 使policy继续输出维持平衡所需的bias action  
因此它类似learned nonlinear PI controller：传统PI是u=Kp*e+Ki*integral(e) Sim2Swim是a=policy(e, integral(e), other state)  

积分状态本身没有单独reward PPO只是通过未来tracking reward学习是否使用它们  
论文没有明确写积分离散更新公式、限幅、anti-windup和reset细节 也没有with/without integral ablation  
所以实验展示了部分trial中没有steady-state error 但没有严格隔离证明误差改善全部来自积分输入  

#### 输出动作和thruster allocation
policy不是direct-to-thruster 输出归一化6维body wrench action：  
a^b = [a_u, a_v, a_w, a_p, a_q, a_r] in [-1,1]^6  
前三维对应surge/sway/heave方向的force 后三维对应roll/pitch/yaw方向的torque  
实际wrench通过tau = K a^b得到 K表示每个DoF允许的最大force/torque  
然后已有的thruster allocation scheme再把body wrench转换为各个推进器命令  

优点是policy不绑定固定thruster数量和布局 换到不同fully actuated AUV时主要替换K和allocation matrix  
缺点是部署仍然依赖准确的thruster allocation和推力上限模型 不是完全忽略机器人硬件结构  
真实实验中速度命令突变时出现小姿态偏差 作者就认为可能是imperfect thrust allocation额外产生了moment  

#### Reward
总reward是正指数型tracking reward和action reward之和：  
r = sum_i r_i + r_q + r_a  
r_i = w_i exp(-||o_i||^2) 用于速度误差/角速度等观测  
r_q = w_q exp(-angle(q_d,q)) 奖励姿态对齐  
r_a = w_a exp(-||a||) 鼓励较小force/torque action  
权重：orientation 0.4 angular velocity 0.05 linear velocity 0.2 action 0.3  
这里action项只能算控制量regularization 不能直接等价成真实推进器energy consumption  
论文式(6)把q_e也写进通用o_i集合 同时又单独定义r_q；因为unit quaternion的norm恒定 这个写法存在不够严谨/疑似符号笔误的问题 真正有意义的姿态reward主要是rotation angle项  

#### Domain randomization
domain randomization延续Learning to Swim的思路 主要随机化：  
robot mass  
robot volume  
center of buoyancy和center of mass之间的offset  
mass和volume从uniform distribution采样 CoB-CoM offset在一个sphere里uniform采样  
目标是覆盖payload变化带来的质量、浮力和静水恢复力矩变化 实现zero-shot sim-to-real  
但正文没有给出这些randomization的具体数值范围 也没有No DR / Small DR / Large DR这种单独消融对比  

#### 真实部署和状态反馈
真实平台是BlueRobotics BlueROV2 Heavy 在indoor pool里验证  
Water Linked A50 DVL测量body velocity并估计水平位置x/y  
BlueRobotics Bar30 pressure sensor测depth z（NED中向下为正）  
BlueROV2 inertial navigation system提供orientation  
3D LOS guidance根据path和position estimate产生linear velocity reference  
policy直接zero-shot部署 没有真实数据finetune或部署后调参  

#### 三组真实实验
1）Figure 4a Straight-line back-and-forth  
AUV沿直线往返 目标yaw和pitch分别由LOS计算的course angle和elevation angle给出 roll接近0  
这组没有随机姿态 姿态主要由路径方向决定  
三个linear velocity components收敛到目标值 heading平滑收敛 论文报告没有steady-state error  

2）Figure 4b Ballast + square path  
机器人左舷增加600g ballast 相当于mass增加约5% 同时改变CM并让机器人从positive buoyancy变成negative buoyancy  
AUV跟踪正方形四个waypoints 所有desired Euler angles都设为0  
surge跟踪较好 sway和heave不能跟上快速reference变化 但最后能收敛  
velocity command突变时姿态出现offset 作者认为可能来自不完美thruster allocation产生的小力矩  
整体没有明显性能退化 用来说明policy对mass/CM/payload变化有一定鲁棒性  

3）Figure 4c Square path + random orientation  
AUV再次跟踪四个square waypoints 但每到一个waypoint切换一次随机姿态setpoint 不是每个control step都随机  
roll和pitch从[-90deg,90deg]采样 yaw从[-180deg,180deg]采样  
surge/sway仍可跟踪 heave response明显较慢  
约37秒处无法跟上较大的desired heave变化 导致depth轻微偏离路径  
作者认为极端roll/pitch使heave方向有效推力下降 推进器利用变得suboptimal  
尽管如此 机器人仍能在极端roll/pitch下保持目标attitude 证明了translation和orientation decoupling的能力  

#### limitation / future work
真实实验仍然只在indoor pool完成 没有暴露在真实海流、波浪、开放水域和长期任务中  
论文没有PID/MPC/Learning to Swim等baseline的同平台定量对比 主要是三组自身验证  
积分增强缺少实现细节和ablation 没有给出anti-windup处理 因而不能把“eliminate steady-state error”理解成严格理论保证  
heave对快速命令的跟踪较弱 极端姿态会降低垂向有效推力  
动作输出是body wrench 部署质量依赖外部thruster allocation；allocation误差会耦合出非期望moment  
DR只覆盖mass/volume/CoB-CoM offset 论文没有展示对强海流、推进器故障、传感噪声或motor delay的系统评估  
任务不是自主导航 没有perception、obstacle avoidance和path planning；真实path following由外部LOS guidance完成  
当前方法面向fully actuated holonomic vehicle 非完整/欠驱动AUV还未验证  
future work：aquaculture和offshore wind-farm等exposed underwater conditions验证 并扩展到non-holonomic systems  

#### 和Learning to Swim对比
Learning to Swim：desired position + desired orientation -> 6 individual thruster commands 是position/pose controller  
Sim2Swim：desired body linear velocity + desired orientation -> 6D body wrench -> thruster allocation 是velocity/attitude controller  
Learning to Swim输入17维 包含position offset和当前/目标quaternion；Sim2Swim输入16维 用velocity/quaternion error并加入两个integral states  
Learning to Swim direct-to-thruster 因而更绑定具体机器人推进器布局；Sim2Swim输出wrench 更方便跨不同thruster配置 但需要额外allocation model  
Learning to Swim在A6000上约10-20分钟训练；Sim2Swim在A2000 8GB上约80秒收敛 3分钟内完成  
Learning to Swim真机重点是position hold + push disturbance rejection；Sim2Swim重点是time-varying velocity tracking + path following + arbitrary attitude  
Learning to Swim真实部署出现sway/pitch steady-state error；Sim2Swim通过integral observation尝试解决长期bias  
Learning to Swim代码isaac-auv-env明确开源；Sim2Swim截至目前未确认代码开源  

#### 对我们的启发
这篇把水下learning-based controller从“到一个pose并保持”推进到“持续跟踪3D速度 同时独立控制姿态”  
系统层次可以写成：  
mission planner / desired path -> LOS guidance -> desired velocity + desired orientation -> Sim2Swim -> body wrench -> thruster allocation -> motors  
因此Sim2Swim比Learning to Swim更适合作为path-following系统的底层velocity controller 但它仍然没有解决高层感知和自主导航  
积分状态是很实用的设计：相比只依靠domain randomization学习平均鲁棒行为 给policy加入error history可以对持续payload/浮力bias产生补偿  
不过更完整的后续研究仍应加入integral ablation、anti-windup、online dynamics adaptation以及开放水域测试  
链接：https://arxiv.org/abs/2512.08656  
IFAC 2026 program：https://ifac.papercept.net/conferences/conferences/IFAC26/program/IFAC26_ContentListWeb_4.html

#### Sim-to-reality adaptation for Deep Reinforcement Learning applied to an underwater docking application
2026 Girona AUV  
发表/收录：arXiv preprint 作者备注currently under review by IROS 2026 暂未确认接收  
作者：Alaaeddine Chaarani, Narcis Palomeras, Pere Ridao，Universitat de Girona / ViCOROB  

这篇不是一般的goal navigation，也不只是把机器人控制到空间中一个没有物理意义的pose。它要求Girona AUV从随机初始位置和yaw接近固定Docking Station（DS），在相机看到3DBM标记后进行相对位姿对准，最后与导向漏斗发生受控接触并插入/落入泊位。  
因此它比Learning to Swim、Fast Policy Learning、Swim4Real等generic pose controller多了task-specific perception interface、碰撞接触和terminal docking success，但底层RL配方仍然很相似：error state -> PPO MLP -> body wrench -> traditional thruster allocation。  

##### 一句话任务定义
固定泊位上的3DBM/外部定位给出AUV相对泊位的位置和yaw误差，policy输出机体系force/torque，使AUV先在水平面完成对中和偏航对准，再下潜进入具有机械导向漏斗的泊位。  

它不做：  
未知环境探索、全局路径规划、一般障碍物避让、从原始图像端到端学习视觉、移动目标追踪。  

它真正增加的难点是：  
对接不是“接近一个点就算成功”，而是必须在有限clearance内对准、容许必要的小接触、限制碰撞冲击并最终完成机械插入。  

##### 完整pipeline
任务链路是：  
USBL或其他外部来源给出泊位粗略位置（首次视觉检测之前）  
-> downward-facing camera检测泊位上的3DBM  
-> 独立视觉模块估计camera-to-dock relative pose  
-> 转换到AUV body frame形成position/yaw error  
-> PPO policy输出6维body wrench  
-> Girona已有的thrust allocation把wrench分配给5个推进器  
-> AUV接近、对准、接触导向漏斗并完成docking  

因此相机不直接接到RL网络。RL看到的是已经估计好的相对位姿，而不是pixels；marker detection和pose estimation仍是传统视觉模块。  

##### 仿真器和digital twin
使用开源海洋机器人仿真器Stonefish建立Girona AUV和Docking Station的digital twin。  
Stonefish包含AUV水动力、传感器接口和刚体碰撞/接触，能够模拟currents、waves和wind；本文只把ocean current作为主要环境扰动，结论部分又把dynamic currents列为future work，因此不能理解成已经系统验证了时变复杂海流。  

泊位模型保留真正影响任务的guiding funnels和碰撞几何，在X/Y方向各有约±25 cm clearance；为降低mesh复杂度，真实泊位外部支撑漏斗的金属框架没有放进训练仿真。  
这比把“到达目标球”当作docking更真实，因为policy必须处理机械接触；但它仍然是简化后的数字孪生。  

作者把Stonefish-RL改成multiprocessing：20个并行训练进程 + 1个evaluation进程，每个线程最高约5倍real-time。  
这远少于Isaac Lab或MJX的数千并行环境，优先保留的是Stonefish的水动力、传感接口和碰撞接触。训练线程headless运行，evaluation线程带可视化。  

仿真和真机都通过同一套ROS sensing/control interface连接policy。这会增加训练通信开销，但减少从simulation切到real vehicle时的软件改动。  

##### 视觉观测在训练中是怎么模拟的
真实机器人使用downward-facing camera检测3DBM并估计泊位相对位姿。  
但是headless并行训练关闭了视觉渲染，因此训练并没有模拟相机图像、光照、浑浊、marker detector和PnP误差。作者只用一个field-of-view / visibility condition模拟“看见或看不见”：  

看得见DS：使用仿真ground-truth relative pose并加入噪声。  
看不见DS或首次检测之前：依赖带不确定性的粗略DS pose，论文假设可来自USBL或其他外部定位来源，并注入更大的occlusion noise。  

噪声尺度随相对平移距离增长，sigma_k = ||e_xyz|| / 6；基础sensor jitter始终存在，目标不可见时再增加一项更大的Gaussian noise。  
所以这里解决的是“带噪相对位姿下的控制”，没有证明policy能处理真实水下图像域差异或marker检测失败。  

##### Observation
policy使用error-based observation：  
noisy relative translation [e_x,e_y,e_z]，在AUV body frame中表示  
yaw error e_psi  
body-frame linear velocity [v_x,v_y,v_z]  
yaw angular velocity omega_psi  
IMU acceleration [acc_x,acc_y,acc_z]  

最关键的限制是：姿态目标只明确使用yaw error，没有roll/pitch target error。  
所以它并不是Learning to Swim或Fast Policy Learning意义上的“任意6-DoF pose regulation”；任务状态本质上是3D position + yaw alignment，pitch/roll更多是机器人动态过程中的自由度。  

##### Action和“6-DoF”的真实含义
连续动作是6维body wrench：  
A = [F_x,F_y,F_z,T_roll,T_pitch,T_yaw]  

policy不直接输出每个推进器的PWM。Girona AUV已有的分配器将desired wrench转换成5个thruster commands。  
由于这个Girona平台只有5个推进器，roll无法被直接驱动。作者仍保留6维action vector，理由是维持general formulation。  

因此论文所称的“6-DoF policy”需要谨慎理解：  
action接口写成6维，不等于该机器人能独立控制6个DoF；roll不可直接驱动，observation/reward又只显式对齐yaw。就本文实际任务而言，更准确的说法是4-DoF docking alignment（x、y、z、yaw）+ 可利用pitch动态完成减速。  

##### Reward和soft docking
总reward：  
R = r_dist + r_angle + r_smooth + r_collision + r_mission  

r_dist：按相对位置误差惩罚，[w_x,w_y,w_z]=[1,1,0.5]，即优先保证X/Y水平对中，再处理Z方向下降。  
r_angle：按yaw error的指数函数奖励朝向对准。  
r_smooth：惩罚相邻时刻action变化，减少高频/激进wrench，帮助真机迁移。  
r_collision：用相邻IMU acceleration变化检测撞击，超过自适应threshold就给-10；threshold在一次碰撞后临时加倍再逐步衰减，避免一次接触因传感器bounce被连续惩罚。  
r_mission：完成docking给+500，episode超时给-10。  

碰撞没有被简单设成episode failure，因为真实对接必然可能接触guiding funnels。reward要学的是“可以借助漏斗完成被动机械校正，但不要高速撞击”，这正是它相对普通pose regulation最有任务含义的设计。  

##### PPO训练
算法是PPO，SAC只在早期被尝试；作者称PPO在水槽实验中更稳定，但没有给出系统的PPO-vs-SAC定量表，因此不能据此得出PPO普遍优于SAC。  
硬件：Intel Core i7 + NVIDIA RTX 4060。  
20个并行training threads，约3小时完成训练。  
physics rate 300 Hz，policy inference和camera refresh均为5 Hz。  
每个episode最长60 s，AUV和DS的初始位置/yaw都会随机化。  
仿真成功率超过90%，训练后mean reward约300-400。  

这个训练速度显著慢于Learning to Swim、Fast Policy Learning和Sim2Swim的分钟级并行训练，但任务包含Stonefish水动力与接触碰撞，不能只按wall-clock横向比较。  

##### “Sim-to-Reality Adaptation”到底做了什么
标题里的adaptation容易令人误以为存在real-data fine-tuning、online system identification或部署时更新网络，但论文没有这些步骤。  

实际的sim-to-real手段是：  
高保真Stonefish水动力和碰撞建模  
随机化AUV/DS初始位置与yaw  
依据距离和可见性注入相对位姿Gaussian noise  
action smoothness与adaptive collision reward  
仿真/真机共用ROS接口  
训练后直接将同一policy放到真机  

因此更准确的术语是zero-shot sim-to-real transfer / robustification，而不是机器学习意义上的online adaptation。  
而且论文没有展示系统性的mass、buoyancy、hydrodynamic coefficient、thruster gain/placement、camera extrinsic等domain randomization。作者反而把thruster position randomization列为future work。  

##### 仿真中学到的行为
作者观察到两个emergent behaviors：  
接近泊位时通过pitch改变艇体迎水面积，利用水阻braking。  
进入导向漏斗时出现小幅yaw oscillation，作者认为这有助于机器人滑入泊位。  

第一个解释在物理上合理；第二个只是基于轨迹观察的推断。论文没有做去掉yaw oscillation的ablation或counterfactual test，因此不能证明振荡必然是有益策略，也可能部分来自控制振荡或allocation误差。  

##### 真实水池实验
平台：真实Girona AUV。  
场地：19 m x 9 m x 5 m test tank。  
泊位：具有真实guiding funnels和外部支撑框架的物理Docking Station。  
感知：downward-facing camera检测真实3DBM并计算相对位置。  
控制频率：5 Hz。  
安全限制：部署时把wrench clipping到AUV最大能力的25%或50%。  

总计10次真实docking mission，8次成功，即80%；论文画出了其中6条轨迹，每次约30-50 s。  
这确实是完整的物理入坞闭环，不是“真机器人对着虚拟障碍物运动”。它验证了真实camera marker pose、真实水动力、真实推进器以及泊位接触。  

但证据规模仍有限：只有10次、2次失败，没有置信区间，也没有按初始距离/姿态分组统计；真实实验没有与PID、MPC或behavior tree做同场定量baseline比较。  
真机还把动作限到训练最大值的25%/50%，说明部署时存在额外手工安全配置，不能把结果理解为完全无人工工程处理。  

##### 与Learning to Swim / Fast Policy / Sim2Swim / Swim4Real的关系
这几篇确实属于高度相似的技术模板：仿真建模 -> error-state observation -> actor-critic policy -> thruster command或body wrench -> domain/noise randomization -> zero-shot水池。论文名字也都在强调swim、fast、sim-to-real等工程卖点。  

真正需要按“policy在系统中控制什么”区分：  

Learning to Swim：目标position + orientation -> individual thrusters；证明direct-thruster 6-DoF pose regulation能够zero-shot下水。  
Swim4Real：目标position + orientation -> individual thruster PWM；任务仍是固定pose regulation，主要增加energy/action分析和更系统的PID对比。  
Fast Policy Learning：目标pose/trajectory -> 6D body wrench -> allocation；主要贡献是JAX/MJX快速训练和PPO/DroQ/SHAC比较，真机依赖Qualisys。  
Sim2Swim：时变3D velocity + arbitrary orientation -> 6D body wrench -> allocation；主要贡献是速度/姿态接口、integral observations和分钟级训练。  
本文Docking：noisy dock-relative position + yaw -> 6D body wrench -> allocation；主要贡献不在新控制算法，而在把policy接入marker-based relative localization、Stonefish contact model、soft-collision reward和真实机械入坞。  

所以你的判断可以概括为：  
控制器骨架确实没有本质变化；任务接口和验证对象发生了变化。  
前四篇主要验证generic pose/velocity regulation，本文开始把这个模板装进一个具有终止条件和物理接触的具体任务。它的增量更像system integration + contact-rich task validation，不是新的RL理论或新的adaptive control机制。  

##### Related work和novelty边界
本文并不是第一篇DRL水下真实入坞。作者自己的related-work table已经列出：  
Bharti et al. 2025：TD3 + AprilTag visual servoing，在BlueROV/test tank做sim-to-real docking。  
Yu and Lin 2025：YOLO light-ring detection + DDPG，在test tank验证。  
Chu et al. 2025：ARSPPO，在真实lake展示docking。  

所以这篇较可信的贡献不是“首次真实RL docking”，而是：  
把Stonefish改造成可多进程RL训练环境。  
为Girona AUV建立包含水动力、marker-relative observation和精细碰撞几何的digital twin。  
通过允许低冲击接触的reward学习垂直入坞。  
在真实Girona AUV和物理泊位上完成8/10次zero-shot实验。  

论文没有与Learning to Swim、Sim2Swim或Swim4Real进行同平台实验，也没有直接对比Fast Policy Learning的controller。虽然related work提到Fast Policy Learning/Tuncay et al.，但那不是docking task的公平benchmark。  

##### 主要limitations
“6-DoF”表述偏宽：真实平台roll不可直接驱动，目标姿态只显式使用yaw。  
“adaptation”不是online adaptation，也没有真实数据finetune。  
训练不是raw-image RL；headless simulation用visibility + noisy ground-truth pose代替真实图像形成过程。  
首次看见marker前依赖USBL或其他外部粗定位，没有解决全局搜索泊位问题。  
仿真泊位省略外部金属框架，真实复杂碰撞几何没有完全复现。  
动态海流、动态泊位、thruster position randomization都仍是future work。  
只有10次真机实验，成功率80%，缺少失败案例分析和统计置信度。  
没有PID/MPC/behavior tree真实baseline；“often destabilize traditional PID or MPC”更多是动机性判断，不是本文实验直接证明。  
没有证明作者所解释的pitch braking和yaw oscillation分别对成功率有多大因果贡献。  

##### 对我们的启发
如果后续研究水下docking或近距离操作，最合理的系统分层不是让一个网络从图像直接包办全部功能，而是：  
粗定位/搜索（USBL、声呐或地图）  
-> 近距离目标检测与relative pose estimation（3DBM、AprilTag、light ring、sonar）  
-> task policy决定对准、下降和接触策略  
-> body-wrench controller / thruster allocation执行  

这篇说明RL可能特别适合传统控制器比较难手工编排的接触阶段，但也暴露了真正困难仍在感知和系统闭环：如果marker不可见、USBL误差过大、海流时变或泊位移动，当前policy并没有完整解决。  

对我们选择研究问题的提醒是：再做一个“error vector -> PPO -> wrench”的通用控制器，创新空间已经很拥挤；更有价值的方向可能是：  
真实声呐/视觉的partial observation与失效恢复  
search-to-approach-to-contact多阶段统一policy  
moving dock / dynamic current下的belief-aware control  
contact-aware sim-to-real adaptation而非仅noise injection  
推进器故障、布局误差和载荷变化下的online adaptation  
与model-based visual servoing / MPC / behavior tree做同平台任务成功率与安全性比较  

链接：https://arxiv.org/abs/2603.12020

#### Towards End to End Motion Planning and Execution for Autonomous Underwater Vehicles Using Reinforcement Learning
2026 University of Haifa, Israel  
作者：Elisei Shafer, Oren Gal  
发表/状态：arXiv v1于2026-06-07公开；截至整理时只确认是preprint，没有正式会议/期刊信息，也没有找到作者公开的完整代码仓库  

这篇是目前和我们想做的“视觉/声呐感知 + 局部导航 + 低层推进器执行”最接近的工作之一。它不再把障碍物坐标或若干理想range直接喂给policy，而是让高层网络读取84x84单目RGB、连续三帧100x100前视成像声呐和本体状态，在未知局部几何中产生短距离相对子目标；另一个低层RL controller再执行这个子目标。  

但是必须先把标题里的end-to-end降到正确的强度：  
它是一个从传感器到推进器的完整hierarchical pipeline，但不是一个联合训练的单网络。  
高层和低层分别训练，中间显式传递相对pose subgoal。  
低层也没有直接输出8个独立推进器，而是输出4个body-axis控制量，再通过作者写死的allocation公式转换为8个thruster values。  
训练和测试全部在HoloOcean里完成，定位、速度和加速度主要来自仿真ground truth oracle，没有真实AUV实验。  

##### 一句话任务定义
给定AUV当前局部的单目图像、前视成像声呐、本体状态以及目标相对位姿，使一台holonomic hovering AUV在静态障碍之间到达目标，同时由学习的低层controller把局部子目标转成推进器控制。  

更准确的任务能力是4-DoF navigation/control：  
x方向surge  
y方向sway  
z方向heave  
yaw转向  

虽然state中写入了完整6-DoF pose和twist，但policy没有独立的roll和pitch action，高层subgoal也只有[x,y,z,yaw]。因此不能把它记成任意6-DoF姿态控制。  

##### 整体hierarchical pipeline
作者把一个navigation MDP拆成两个独立MDP：  

High-Level policy，2 Hz：  
RGB + stacked FLS + proprioception + relative final goal  
-> 输出相对当前机体系的局部subgoal [delta_x,delta_y,delta_z,delta_yaw]  

Low-Level policy，10 Hz：  
proprioception + desired subgoal + achieved goal  
-> 输出[Tx,Ty,Tz,T_yaw]  

Fixed thrust allocation：  
[Tx,Ty,Tz,T_yaw]  
-> 按固定线性组合变成8个推进器命令  

这种分层本身是合理的：高维视觉/声呐不必以10 Hz以上重新规划，低层controller可以更快稳定执行；而且高层每次只提出最多1.9 m的局部动作，避免直接规划任意远的终点。  

但两层不是joint training：  
先在无障碍open water单独训练low-level policy并冻结best policy。  
再用已经训练好的low-level controller作为执行器训练high-level policy。  
所以不能证明视觉encoder、subgoal representation和thruster controller经过全局任务回报共同优化。  

##### 仿真器和AUV
使用HoloOcean + Unreal Engine，RL接口由作者自定义Gymnasium wrapper连接。  
机器人是HoloOcean中的8-thruster hovering AUV，属于fully actuated / holonomic平台，能够sway而不必像torpedo-shaped AUV一样先前进再转弯。  

训练场景只来自OpenWater scenario中的一个固定位置：  
一个cube  
一个cone  
一个cylinder  
固定starting point  
三个随机目标区域  

目标区域被布置在障碍物之间，作者希望机器人从中穿过，而不是直接上浮越过全部障碍。  
这比二维圆障碍的range-only仿真更接近感知导航，但环境多样性仍然非常低：没有自然地形、珊瑚、管线、悬浮物、动态障碍、海流或不同场景随机布局。  

论文没有详细列出HoloOcean vehicle使用的水动力参数、推进器动态、current设置或domain randomization。仅仅使用HoloOcean并不能自动说明这些sim-to-real因素已经被验证。  

##### High-Level observation
高层proprioceptive state共21维：  
6维body linear/angular velocity nu=[u,v,w,p,q,r]  
3维linear acceleration  
12维desired goal state，其中包含当前机体系下的目标相对pose以及desired zero twist  

外感知输入：  
单帧84x84x3 monocular RGB image  
最近三帧100x100 FLS image stack，得到100x100x3  

声呐叠三帧是为了让policy从时序变化判断自身运动和障碍接近趋势；RGB只输入当前单帧，没有recurrent memory。  

作者称这些是raw inputs，但严格说是minimally processed input：  
相机和声呐已经被resize到很低分辨率  
声呐做了三帧stack  
视觉通过pretrained ResNet-10转成feature  
训练时用DrQ image augmentation  

更重要的是，目标相对位置、body velocity和acceleration都由仿真oracle直接提供，并没有运行DVL/IMU/EKF/SLAM。  
所以“无地图”不等于“无定位”：策略仍然假设每一步都准确知道目标在当前机器人坐标系中的相对pose。  

##### Low-Level observation
低层state写成39维：  
当前6-DoF pose eta  
6维linear/angular velocity nu  
3维linear acceleration  
12维desired goal state  
12维achieved goal state  

desired goal在以起点为中心、边长4 m的cube中采样，也就是位置各轴大约在[-2,2] m；desired twist固定为0。  
高层subgoal的位置上限设为1.9 m，基本就是让高层始终给低层分布内的小目标。  

低层依赖完整pose和goal state，但这些也都是仿真真值。论文没有验证真实水下位置漂移、DVL dropout或goal estimator误差长期累积后是否还能工作。  

##### Action和推进器分配
高层连续动作：  
a_HL=[delta_x,delta_y,delta_z,delta_yaw]  
位置限制在[-1.9,1.9] m，yaw在[-pi,pi]。  

低层连续动作：  
a_LL=[Tx,Ty,Tz,T_yaw] in [-1,1]^4  

它们不是8个individual-thruster commands，而是归一化的surge/sway/heave/yaw控制量。作者随后手工分配：  
T1,T2,T3,T4全部等于Tz  
水平四推进器T5-T8由Tx、Ty和T_yaw按照正负号线性组合得到  

因此论文从系统输入输出角度确实最终驱动8个推进器，但低层policy没有学习thrust allocation，也不能自动适应推进器布局、安装角度、单推进器故障或asymmetric gain。  
它和Sim2Swim的body-wrench -> allocation思想更接近，而不是Learning to Swim那种6个individual thruster直接由policy输出。  

##### 两层分别怎么训练
Low-Level：  
Stable-Baselines3 SAC + HER  
无障碍open water  
随机位置和yaw目标  
150k simulation steps  
论文称相当于约4小时simulation time，但没有给清楚实际wall-clock  
20个随机目标测试全部成功  
成功阈值是position error小于0.2 m且heading error小于0.2 rad，约11度  

High-Level：  
修改SERL framework接入HoloOcean  
算法使用RLPD，也就是Reinforcement Learning from Prior Demonstrations  
off-policy训练时同时从human demonstration buffer和online replay采样  
pretrained ResNet-10做图像feature preprocessing  
DrQ做图像augmentation  
单张RTX 4000 16 GB训练  

训练前由人类操作员采集80条demonstrations。操作者同时观看RGB、FLS和目标相对位置提示，用点击界面选择局部相对位置，再用键盘控制depth和yaw；这些动作正好对应高层的4维subgoal。  

论文没有报告高层训练用了多少environment steps、多少wall-clock、80条demo总时长、replay ratio、最终网络结构细节或训练曲线。  
因此摘要里“sample-efficient、可望在真机online训练”的定位目前主要是动机，并没有被完整定量支持。  

##### Reward
Low-Level reward是goal-conditioned sparse reward：  
每个timestep给-1  
到达position 0.2 m和orientation 0.2 rad阈值内给+1  
HER把实际到达的state重新作为goal，提高稀疏奖励训练效率  

High-Level reward是dense reward：  
负的goal distance  
负的desired-angle error  
到目标1.9 m范围内给+1  
碰撞给较大惩罚  

这里有两个复现问题：  
没有写出high-level总reward的完整公式和各项权重。  
1.9 m到达阈值相对局部任务并不小，而且实验部分没有明确说明最终成功率是否也使用相同阈值。  

##### 训练时的传感器真实性
训练阶段所有simulated sensors都没有noise。  
pose、velocity和localization直接读取simulator perfect ground truth。  
RGB只使用HoloOcean渲染图像。  
FLS使用HoloOcean imaging-sonar simulation。  

训练后才做两种stress test：  
1）给pose、IMU、DVL和FLS加入人工噪声  
2）把HoloOcean fog从训练时0.03提高到0.1，并给FLS之外的传感器加噪声  

噪声设置包括：  
position sigma 0.1 m  
rotation sigma 0.0001 rad  
linear acceleration sigma 0.012 m/s^2  
angular velocity sigma 0.0001 rad/s  
DVL velocity sigma 0.05 m/s  
sonar additive sigma 0.05、multiplicative sigma 0.1、range sigma 0.025 m  

这个实验说明policy对这些人为扰动有一定容忍度，但不是sim-to-real：  
没有训练时sensor domain randomization  
fog只是视觉退化的简化proxy，不等于真实水下吸收、散射、backscatter和灯光变化  
声呐噪声模型不能覆盖真实multipath、shadow、surface/bottom reverberation和不同材质反射  
没有真实相机、真实FLS或真实DVL数据  

##### Seen / unseen实验结果
作者在同一个场景里定义seen goal areas和unseen goal areas。  
这里的unseen不是一张全新的地图：仍是同一个cube、cone和cylinder，只是目标位置和观察障碍物的方向没有进入训练数据。  

无噪声结果：  
seen near cube 20/20  
seen behind cone 19/20  
seen near cylinder 20/20  
unseen near cube 12/20  
unseen behind cone 20/20  
unseen near cylinder 2/20  

最严重的near-cylinder OOD失败来自机器人需要绕过训练时没有从该角度见过的曲面/end-cap。policy会卡入local minimum。  
因此这篇最诚实也最重要的结果不是“已经实现通用视觉导航”，而是：在只含三个几何体的小场景里，end-to-end感知动作映射仍然会对一个未见视角/曲面发生灾难性泛化失败。  

加入传感噪声：  
seen为20/20、18/20、20/20  
unseen为12/20、20/20、5/20  

增加fog和非FLS噪声：  
seen三组均22/22  
unseen为8/20、22/22、8/20  

near-cylinder从2/20偶然升到5/20或8/20并不说明感知变差反而更好。作者自己的解释是噪声偶尔把policy从local-minimum attractor里推出来，基础泛化问题仍然存在。  

##### RRT* + PD baseline
传统baseline是：  
把完整二维障碍地图预先给RRT*  
RRT*离线生成一串2D waypoints  
HoloOcean PD position controller依次跟踪  

RRT*运行5000 iterations，障碍物额外膨胀1 m以补偿机器人尺寸和PD tracking error。  
在地图和真实切片比较一致的场景里，RL轨迹大约比RRT* + PD长1%-6%；摘要和discussion概括为seen约4%、novel约6%。  

这个比较只能说明学习policy在已见简单场景中没有绕太大的远路，不能证明它接近全局最优：  
RRT*拥有完整2D地图，RL只有局部传感，信息条件不相同。  
RRT*只规划2D且不规划yaw/depth，RL action却包含z和yaw。  
RRT*地图用障碍底部最大截面，cone在某些深度被表示得过大，导致RL甚至能走过地图所标的障碍区域。  
两套方法的控制器也不同：RRT*配PD，本文配learned LL policy。  

另外Table II还残留“PF Placeholder”命名且正文没有清楚解释，表格bottom-half划分和这些行的关系不够规范。论文目前属于较早期preprint，结果表仍有明显整理空间。  

##### 它到底算不算end-to-end
可以从两个层次回答：  

系统I/O意义上：基本算。  
运行时从RGB/FLS/proprioception开始，最后确实生成8个推进器值，中间不显式建图，也不运行传统local planner。  

学习优化意义上：不是严格end-to-end。  
高层与低层分开训练。  
中间存在人工定义的4维relative-pose subgoal bottleneck。  
最终存在人工固定的thrust allocation。  
目标相对pose和本体状态来自oracle，而不是从raw sensors学习出来。  

因此最准确的描述是：  
hierarchical sensor-to-actuator learning pipeline  
而不是single-policy raw-perception-to-individual-thruster learning。  

这种“不是单网络”并不一定是缺点。对于真实AUV，高低层分开反而更容易训练、调试和做安全限制；问题在于论文的标题和贡献表述需要把这个工程分层说清楚。  

##### 有没有做有效ablation
几乎没有围绕方法组成做消融。论文没有比较：  
RGB-only vs FLS-only vs RGB+FLS  
单帧FLS vs三帧stack  
RLPD vs SAC/PPO或纯behaviour cloning  
有无80条human demonstrations  
ResNet-10 pretrained vs from scratch  
DrQ augmentation有无  
learned low-level vs HoloOcean PD controller  
hierarchical policy vs直接sensor-to-thruster policy  
oracle localization vs带漂移/丢失的state estimator  

因此现在不能判断成功主要来自RGB、FLS、human demos、SERL/RLPD，还是低层controller。  
Noise和fog测试属于robustness evaluation，不是对核心模块贡献的ablation。  

##### 与Nav2Goal、Learning to Swim和Sim2Swim的关系
Nav2Goal：front RGB + relative goal -> 离散yaw/pitch，高层视觉反应式导航；通过conditional imitation learning训练，有真实珊瑚礁约1 km部署。它没有学习low-level thruster control，但真实视觉与真实障碍证据远强于本文。  

Learning to Swim：pose/state -> 6 individual thrusters，解决低层6-DoF pose regulation并zero-shot真机；没有障碍感知。  

Sim2Swim：desired velocity/orientation -> 6D body wrench -> allocation，解决低层速度/姿态跟踪；也没有障碍感知。  

本文：RGB + FLS + state + relative goal -> learned subgoal -> learned 4-DoF controller -> fixed allocation。它的增量是把感知导航和低层执行放进同一条learning pipeline，但真实感知与真实执行都尚未验证。  

IROS 2019的End-to-end Sensorimotor Control Problems of AUVs已经使用视觉/sonar做过水下DRL sensorimotor control，Nav2Goal也已完成真实visual goal navigation。作者更可信的novelty边界是：他们没有找到此前把minimally processed monocular RGB和raw imaging-sonar frames一起输入hierarchical DRL、最终闭环到thruster values的工作，而不是“第一篇水下end-to-end导航”。  

##### 主要limitations
纯HoloOcean仿真，没有水池、湖泊或海试。  
所有训练sensor均为noise-free，定位和速度来自oracle。  
没有处理真实水下相机折射、吸收、散射、backscatter和灯光域差异。  
没有真实imaging-sonar数据或sim-to-real声呐验证。  
训练只含一个固定场景、三个基础几何体和固定起点，dataset diversity极低。  
所谓unseen area仍在同一地图中；遇到未见曲面视角时成功率只有2/20。  
动作只有x/y/z/yaw四个DoF，不是完整6-DoF attitude control。  
低层输出4个聚合控制量并固定allocation，不是learned individual-thruster control。  
高层、低层不是joint end-to-end optimization。  
没有current、dynamic obstacle、thruster fault、payload变化或动力学DR。  
没有关键sensor/algorithm ablation。  
没有报告高层training steps、wall-clock、训练曲线和demo总时长。  
没有collision rate、minimum clearance、能耗、动作平滑度或真实计算延迟指标。  
RRT* baseline在地图、维度和controller上与RL不完全公平。  
不同表格有20和22次测试，未给置信区间或显著性检验。  
未确认代码开源，复现性目前有限。  

##### 对我们的判断和启发
这篇证明了我们想做的方向不是完全空白：到2026年已经有人明确提出RGB + FLS + proprioception -> local subgoal -> thruster execution，并在HoloOcean完成概念验证。  

但它也反过来帮助我们把真正的research gap说得更精确：  
不是“首次在仿真里把相机接到RL避障”。  
而是“如何让这种multimodal sensor-to-actuator policy在真实水下感知退化、真实定位误差和真实水动力下可靠工作，并对未见自然几何泛化”。  

如果我们沿这个方向做，至少应该超过它的几项关键弱点：  
真实RGB + real/sim FLS对齐，而不是只做ideal rendered sensors。  
训练场景大规模procedural randomization，包含自然障碍、纹理、光照、浑浊和声学材质变化。  
actor只能使用可部署的RGB/FLS/IMU/DVL/depth/history，perfect depth/occupancy只给privileged critic或teacher。  
加入recurrent belief处理partial observation与sensor dropout。  
使用真实relative-goal estimator并注入drift/dropout，而不是oracle pose。  
至少做RGB-only、FLS-only、fusion、no-history、no-demo和different low-level controller消融。  
最终在真实物理障碍中闭环验证，而不是实物机器人面对虚拟障碍。  

从系统设计上，我们不必执着于单网络。一个更有机会落地的架构仍然可以是：  
raw RGB/FLS/history -> learned local navigation policy -> desired body velocity或subgoal -> robust low-level controller -> allocation/thrusters。  
只要训练和实验清楚证明原始感知确实改变实际机器人运动并避开真实障碍，这种hierarchical end-to-end比把所有问题硬塞进单个policy更可信。  

链接：https://arxiv.org/abs/2606.08513  
本地PDF：papers/underwater/Shafer_2026_Towards_End_to_End_AUV_RL.pdf  

#### CORAL-AUV: CFD Oriented Reinforcement Learning for Autonomous Underwater Vehicles
2026 MIT-WHOI Joint Program / Woods Hole Oceanographic Institution / MIT Aerospace Controls Lab  
作者：Steven Roche, Milo Van Mooy, Nathan McGuire, Levi Cai, Jonathan P. How, Yogesh Girdhar  
发表/状态：arXiv v1于2026-07-10公开，16页；截至整理时未确认正式会议或期刊收录  

这篇的RL任务本身仍然是6-DoF目标pose / waypoint regulation：给机器人目标位置和姿态，PPO直接输出CUREE的6个推进器PWM命令，让机器人在规定时间内到达目标。  
它不做视觉避障、未知环境导航或路径规划。真实实验中的U形和方形路径只是外部依次发送几个waypoints，policy负责执行每段低层运动。  

它相对Learning to Swim的真正变化不是新的policy结构，而是追问一个更基础的问题：  
如果训练仿真器里的水动力阻力模型结构本身不正确，domain randomization能不能弥补？  
作者保留Learning to Swim的Isaac Sim + PPO + direct-to-thruster框架，只替换仿真器计算drag force / torque的方法，再将不同policy部署到同一真实CUREE比较。  

##### Drag和drag model是什么
drag是机器人相对水运动时，水对机器人产生的阻力和阻力矩。它不只是“向后拉”的一个标量，而是六维wrench：  
W_drag = [F_x,F_y,F_z,tau_roll,tau_pitch,tau_yaw]  

drag取决于：  
机器人相对水的线速度v和角速度omega  
机器人形状、朝向和表面几何  
水的密度、黏度与流动状态  
不同轴之间的耦合  

drag model就是仿真器中的函数：  
W_drag = f(v, omega, orientation, geometry, fluid parameters)  
仿真器在每个physics step调用这个函数，把得到的水动力加入机器人运动方程，再算下一时刻的速度和位姿。  

最简单的diagonal drag model把每个轴独立处理，例如：  
F_x = -d_x v_x - q_x |v_x|v_x  
F_y = -d_y v_y - q_y |v_y|v_y  

这隐含假设x方向速度只产生x方向阻力。但真实AUV有推进器、支架、空腔、相机和不规则外形，向前游时也可能产生侧向力、垂向力和pitch/yaw moment：  
v_x不仅影响F_x，也可能影响F_y、F_z和tau。  
这种cross-axis coupling正是简单惯性盒模型很难表达、CORAL-AUV重点研究的部分。  

##### CFD是什么
CFD是Computational Fluid Dynamics，计算流体力学。它不是一种RL算法，也不是一个普通的经验阻力公式。  
它把机器人CAD几何周围的水域划分成大量网格，在这些网格上数值求解流体的质量守恒和动量方程，得到机器人周围的流速、压力和剪切应力，再沿机器人表面积分出总force和torque。  

相对简单drag equation，CFD可以表达：  
不规则艇体和appendages如何改变水流  
不同方向速度产生的cross-axis force / moment  
局部分离、尾流、压力分布和湍流的影响  

但CFD非常昂贵。一次流场求解可能需要几十秒到数分钟，而PPO需要并行运行数百万个physics steps，所以不能在每一个RL step里直接运行OpenFOAM。  

##### CFD是否开源，以及是否“完全不简化AUV”
CFD是一类计算流体的方法，不能简单说“CFD开不开源”；本文使用的具体软件OpenFOAM是GNU GPL许可的免费开源CFD工具：https://www.openfoam.com/  

相对equivalent inertia box，作者使用CUREE的CAD model，确实保留了更多真实艇体、框架和appendage外形，不再把整个AUV近似成一个盒子。  
但这不等于完全无简化地重现真实水动力：  
CAD表面仍要离散成有限分辨率mesh，论文没有给mesh-convergence study或说明保留了哪些小零件  
使用RANS k-omega SST turbulence model，而不是解析所有瞬时小尺度涡旋  
只求steady-state wrench，不建模启动、急转、尾涡历史和fluid-memory transient effects  
linear motion与angular motion分别采样，最后将两个surrogate输出相加，没有完整联合建模f(v,omega)  
没有说明显式解析每个propeller旋转、thruster jet、喷流与艇体/推进器之间的流场相互作用  

因此准确说法是：使用较完整CAD和流体方程得到比box model更真实的稳态耦合drag，但仍是经过湍流、网格、稳态和运动分解简化的高保真近似，不是真实水本身。  

##### Surrogate drag model是什么
surrogate model是一个便宜的近似器。本文先离线用OpenFOAM生成大量“速度 -> 六维水动力”的CFD数据，再训练MLP模仿这个映射：  

CAD + sampled velocity/orientation  
-> OpenFOAM CFD  
-> [linear/angular velocity, six-dimensional wrench] dataset  
-> train neural surrogate drag models  
-> embed surrogate MLPs into Isaac Sim  
-> train PPO policy rapidly  

RL训练阶段已经不再运行OpenFOAM；每个physics step只做一次小型MLP inference，因此能保留部分CFD耦合特征，同时把policy训练控制在约15分钟以内。  

##### 三种drag model不等于训练三个drag MLP
“三个drag model”表示三种不同的水动力计算方式，不表示三个神经网络：  
inertia-box drag：解析公式，不训练MLP  
System-ID drag：用真实coast-down数据拟合对角阻力系数，仍是公式，不训练MLP  
CFD surrogate drag：训练2个MLP，一个输入linear velocity，一个输入angular velocity，二者都输出六维wrench并在运行时相加  

之后作者在三种不同physics environment中分别训练PPO：  
inertia-box environment -> PPO policy A  
System-ID environment -> PPO policy B  
CFD-surrogate environment -> PPO policy C  

最后部署到同一真实CUREE进行对比。因此需要区分两层网络：CFD surrogate MLP负责模拟“虚拟水”，PPO policy负责根据观测输出推进器命令。  

##### CFD使用什么速度和水流参数
物理上drag应由机器人相对水体的速度决定：  
v_relative = v_robot - v_water  

CFD数据生成需要指定水的density、dynamic viscosity、来流速度/方向、CAD、姿态、网格、边界条件和turbulence model。本文默认参数包括fluid density 997 kg/m^3、dynamic viscosity 0.001306 Pa*s。  

作者随机采样AUV的linear/angular velocity与orientation。典型CFD实现可以把AUV固定，让水以相反方向流过它；对稳态drag而言，这等价于AUV以相同相对速度穿过静水。  

论文在Isaac Sim中把当前body-frame linear velocity v与angular velocity omega输入surrogate。其训练设置基本把周围水体当成静止背景，所以数值上：  
v_water = 0 -> v_relative = v_robot  

如果真实海流与机器人同向，真实relative velocity会小于DVL/对地速度；逆流时会更大。本文没有显式的current estimator、water-relative velocity observation或时变current-conditioned drag model，因此真实海流主要被当作未建模外部扰动，由闭环policy看到tracking error后再反应式补偿。  

##### CFD、Isaac Sim训练和真机部署不是同一过程
阶段1 CFD data generation：输入CAD、流体参数和采样速度，离线得到velocity -> wrench标签。  
阶段2 surrogate training：用CFD标签训练2个drag MLP。  
阶段3 PPO training：Isaac Sim每个step调用drag MLP生成虚拟水动力，并更新模拟AUV；只有这一阶段训练control policy。  
阶段4 real deployment：冻结PPO参数并放到CUREE；真机不运行OpenFOAM，也不运行surrogate drag MLP，真实水自然产生真实drag。  

所以sim和real不是“进行一样的训练”，而是same frozen policy、different plant：simulator用surrogate产生近似水动力，真实机器人由真实水产生水动力。surrogate的唯一任务是让训练时的虚拟水更像真实水。  

##### 三种drag model对照
1）Equivalent inertia box  
根据真实机器人的mass和inertia，把机器人近似成具有相同惯性特征的box，再用MuJoCo / Learning to Swim式解析模型计算阻力。速度快，但本质上是简化的diagonal drag model。  

2）System ID  
在真实CUREE上进行coast-down实验：先沿某一轴加速，关闭推进器，让机器人自由减速，再使用IMU、DVL和EKF速度拟合阻力系数。  
作者采集60次平移实验：±x、±y、±z各10次；旋转实验计划30次，1次yaw数据损坏，最终使用29次。  
虽然系数来自真机，但仍然逐轴独立拟合，是linear diagonal model，不能表达cross-axis coupling。  

3）CFD surrogate drag model / SDM  
使用CUREE CAD模型和OpenFOAM，采用steady-state RANS k-omega SST turbulence model与PIMPLE solver，随机采样机器人orientation、linear velocity和angular velocity并计算六维wrench。  
生成约183,000组linear-velocity数据和约90,000组angular-velocity数据。  
分别训练linear drag MLP和angular drag MLP，每个网络3层、每层1024 units，ReLU + Adam + MSE，数据按80/10/10划分；运行时将两个网络输出的wrench相加。  

因此三种模型比较的不是三种RL算法，而是：  
相同PPO + 相同任务 + 相同机器人，只改变training simulator里面水怎样“推/拉”机器人。  

##### Observation、Action和PPO训练
每个训练环境中的机器人在目标5 m范围内以随机pose初始化，要求10 s内到达目标position和orientation。  
使用Isaac Sim并行10,000个环境，PPO policy训练约15分钟以内。  

Observation：  
target orientation  
current orientation  
body-frame linear velocity  
body-frame angular velocity  
goal position offset  

Action：  
policy直接输出CUREE的6个normalized PWM / thruster commands，不是Sim2Swim或Fast Policy式body wrench。  

所以它的控制接口最接近Learning to Swim；CFD surrogate只存在于训练仿真器的environment dynamics中，真实部署时不需要运行CFD或surrogate。部署时只运行PPO policy。  

##### Domain randomization
随机化vehicle volume和center-of-buoyancy / center-of-mass offset：  
普通外场实验：volume ±1%，CoB-CoM offset在半径0.01 m sphere内随机  
模拟尾部增加1 lb：同样使用±1%和0.01 m  
模拟尾部增加2 lb：volume ±2%，offset radius 0.02 m  

这里的DR范围不是随意取的，作者根据在尾部增加配重后预计产生的浮力/重心变化来设计。  
但正文没有把mass列为随机变量，而主要用volume与CoB-CoM offset近似额外载荷效应，这仍是一个简化。  

##### 两种reward和实验任务
1）Visual-survey reward / 外场U形路径  
用于珊瑚礁视觉survey。外部依次给出U形路径的corners，每段4 m；不要求在corner达到任意完整姿态，但奖励机器人面向下一个目标并保持upright。  
reward包括position、facing/yaw、stability、linear/angular velocity和action / thruster usage项。  

2）General 3D pose reward / 水池U形任务  
同样依次跟踪U形waypoints，每段1.7 m，同时需要满足position和orientation threshold。  
reward简化为position、orientation、velocity和action四项。  
虽然论文称general 3D pose，真实实验的target orientation只改变yaw，以保证DVL保持bottom lock；没有像Sim2Swim那样让机器人执行极端roll/pitch。  

##### Reward shaping实验
作者设计三组reward coefficients：  
Conservative：偏保守，动作和速度惩罚较强  
Balanced：提高pose attainment权重  
Aggressive：减少action penalty，鼓励更快运动  

这部分不是完全离线确定的。作者先将一组policy放进水池，根据真机表现再设计下一组系数，最后为每种drag model选择表现最好的配置，然后加入2 lb对应的DR重新训练。  

所以单个最终policy从仿真到部署是zero-shot，但整个controller development使用了多轮真实水池测试进行reward tuning，不能理解成开发全过程完全不接触真机。  

##### 外场实验
使用CUREE AUV，在Lameshur Bay, St. John, USVI进行真实海域U形路径测试。  
状态估计主要由IMU + DVL + EKF提供，看到海底AprilTag时再融合tag measurement。  
每种drag model对应的policy各测试5次，比较：  
mean cross-track error  
waypoint completion time  
thruster effort = normalized PWM commands平方和  

相对inertia-box policy，CFD policy的主要结果为：  
cross-track error约低19%  
waypoint traversal约快11%  
thruster effort约低31%  

论文把最后一项称为energy improvement，但它没有测量电池端真实电压、电流或能量，只用PWM平方和作为proxy，所以更严谨的说法是control/thruster effort降低31%。  

作者还在更暴露、具有真实科研价值的Yawzi Reef约7 m深处进行了box-pattern演示。该区域有约10-20 mph surface wind产生的波浪，证明policy至少能在受控水池之外运行。  

##### 水池、reward和drag model对照
在水池3D pose任务中，inertia-box policy对reward系数非常敏感：某些reward下几乎不移动或无法完成yaw，只有Balanced配置较好。  
System-ID policy在三组reward下表现中等。  
CFD policy跨三组reward的迁移最稳定；选择各模型最好的policy比较时，它完成waypoints更多、时间更短且PWM effort更低。  

这说明高保真模型的价值不只是提高最终一个policy的分数，还可能使simulator中的reward设计更能预测真实硬件行为。  

##### 2 lb载荷扰动实验
作者在CUREE尾部左右各增加1 lb dive weight，总计2 lb / 约0.91 kg，并用2 lb对应的DR分别训练三种policy。  

真实部署结果：  
CFD+DR policy：5次完整U形任务全部成功，共完成15个waypoints  
inertia-box+DR policy：未完成任何waypoint  
System-ID+DR policy：未完成任何waypoint  

CFD policy加配重后比无配重时更慢、PWM effort更高，但仍能完成任务。  
尤其值得注意的是System-ID policy在仿真中的reward看起来很好，却完全没有迁移成功；CFD simulation的reward才正确预测了zero-shot transfer。  

本文最重要的结论不是“DR有效”，而是：  
如果drag model缺少真实存在的cross-axis coupling，仅仅扩大domain randomization范围不一定能修复结构性模型错误。  

##### 这篇做了哪些有效对照/消融
相对刚才的Docking论文，这篇确实围绕核心claim设计了对照：  
三种drag model：inertia box / real-data System ID / CFD surrogate  
三组reward shaping：Conservative / Balanced / Aggressive  
不同DR强度：No DR / 1 lb / 2 lb  
正常机器人与尾部增加2 lb  
仿真reward与真实transfer success的对应关系  
受控tank与真实reef field deployment  

这不是神经网络层数之类的局部消融，而是system-level ablation：保持PPO和任务基本不变，只改变simulation physics fidelity，再比较真实部署。它比“只报告自己的success rate”更能支持论文结论。  

不过仍缺少：  
CFD surrogate与真实force/torque ground truth的系统误差表  
steady-state CFD与transient CFD对比  
PID/MPC同平台任务对比  
任意roll/pitch的完整6-DoF真实测试  
真实电功率/电量测量  
完整CFD数据生成时间和计算资源统计  

##### 与Learning to Swim系列的关系
Learning to Swim：证明简化inertia-box dynamics + DR + PPO direct thrusters可以zero-shot完成6-DoF pose regulation。  
Swim4Real：仍做pose regulation，重点转到PWM平滑和energy proxy，并与PID比较。  
Fast Policy Learning：改用MJX + body wrench接口，重点是训练速度和RL算法比较。  
Sim2Swim：改成time-varying velocity + attitude接口，引入integral observation。  
CORAL-AUV：控制任务与policy接口退回/延续Learning to Swim，但把研究重点放到simulation dynamics fidelity，直接检验drag model如何影响真实迁移、能效和payload robustness。  

发展关系可以概括为：  
Learning to Swim问“简单仿真 + DR能不能让6-DoF RL controller下水？”  
CORAL-AUV进一步问“当简单仿真的drag结构错了，DR是否仍然够用？”  

本文给出的答案是：对普通配置可能够用，但在reward变化和2 lb OOD payload下不够；包含耦合水动力的CFD surrogate更可靠。  

##### 主要limitations
policy任务仍然是目标pose / waypoint tracking，没有perception、obstacle avoidance和autonomous planning。  
“15分钟训练”只计算已有surrogate之后的PPO训练，没有报告生成约273,000组OpenFOAM数据需要的总wall-clock和算力；换机器人CAD后通常需要重新生成CFD数据。  
CFD是steady-state，不含流体memory、vortex history和真实transient effects。  
linear velocity与angular velocity被分别采样和建模，最后直接相加，降低CFD数据维度的同时忽略了线运动与转动之间的联合高阶耦合。  
System-ID baseline是线性对角模型，且coast-down数据可能不覆盖推进器工作时的速度范围；它失败不能说明所有system-identification方法都不如CFD。  
reward是在多轮水池实验后调整的，整个开发流程不是严格的no-real-data zero-shot。  
实验所称energy是PWM平方和proxy，不是真实功耗测量。  
真实姿态目标主要只改变yaw，完整任意roll/pitch的6-DoF实验证据有限。  
没有在本文任务中与PID或MPC做公平的真实同平台baseline。  

##### 我们的判断和启发
这篇比“再换一个PPO reward做定点控制”更有实质增量，因为它把sim-to-real gap从笼统的domain randomization口号拆成可以实验检验的physics model mismatch。  
它的RL算法没有创新，任务也没有超出position/waypoint tracking；真正贡献是CFD-to-surrogate-to-RL pipeline，以及三种drag assumption在真实CUREE、配重扰动和珊瑚礁外场中的系统比较。  

对后续研究最重要的启发：  
domain randomization只能在模型族覆盖真实动力学时发挥作用；如果仿真器根本不包含cross-axis coupling，随机化几个diagonal coefficients不能创造缺失的物理结构。  
CFD surrogate是一种介于纯解析模型和直接CFD之间的折中：前期数据生成昂贵，但训练/部署快，适合外形复杂、payload变化明显、又需要反复训练controller的AUV。  
更进一步的方向是把CFD prior、少量真实数据和online residual adaptation结合，而不是每换一个payload都重新生成大规模CFD。  

链接：https://arxiv.org/abs/2607.09557

### 初步判断
水下learning-based导航和控制目前可以分开看：  
视觉导航类：Nav2Goal这种端到端视觉反应式policy 重点是goal conditioning / imitation / waypoint reaching  
mapless navigation类：Hadi TD3 / BlueROV2 / HUAUV这类用range/distance/occupancy/raycast + RL做局部避障到目标 其中多数仍是理想化感知下的simulation-only验证  
低层控制类：Learning to Swim / Sim2Swim / Fast Policy Learning / CORAL-AUV 这类从目标pose或velocity与当前状态生成thruster-level action或body wrench 做6-DoF控制和sim-to-real；CORAL-AUV进一步把研究重点推进到training simulator的水动力模型真实性  
多模态sensor-to-actuator类：Towards End to End Motion Planning这篇开始把RGB、FLS、相对目标、高层subgoal和低层thruster execution放进同一条hierarchical learning pipeline，但目前只有三个基础几何体的HoloOcean仿真、oracle localization且无关键ablation，不能视为真实水下端到端导航已经解决  

如果我们的兴趣是“水下机器人learningbased导航和运动控制” 可以先沿两条线整理：  
1）高层导航：当前图像/距离传感/目标 -> yaw/pitch/velocity command  
2）低层控制：目标pose/速度命令 -> thruster command  
暂时不进入水下建图 只把VO/EKF看成提供relative goal或训练标签的工具  
