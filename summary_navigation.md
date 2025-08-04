# Learning a State Representation and Navigation in Cluttered and Dynamic Environments
![](./navigation/hutter.png)

为了让网络学习滤波器的效果 用了一个vae结构 将过去几帧的图像过一个encoder 得到一个latent 然后过一个decoder 得到一个重构的图像 然后计算重构图像和被真实滤波器滤波的原图像作loss  训练强化学习的时候图像vae冻结  

encoder的输入 加上相机位置过LSTM 得到的hidden state 会过一个decoder去估计下一时刻的vae latent 与原latent作loss 这个hidden state 就作为状态表示和goal一起作为输入 让RL去学习 仿真环境为gym 奖励比较常规 但是depth image 用了仿真和真实数据 
LSTM也是一个ae结构 decoder 会同时估计过去的和未来的 过去的作loss 
# iPlanner: Imperative Path Planning  
RSS（2023）  
预先建高精度图的问题已经解决的比较好了  
传统规划方法：分层涉及到模块延迟的问题 而且效果层级制约  
端到端的方法（有标签）：label工作量大 依赖于数据集  
端到端（无标签）：RL带sensory后本身的采样慢 收敛慢 给一些额外先验或者辅助任务后又容易影响任务的收敛方向 
![](./navigation/iplanner.png)
pipeline非常清晰 终点目标坐标过一个mlp 深度图过一个cnn（resnet）concat一下输入到planning network 得到一个n*3的路径点序列 这个序列会被TO优化 在每两个路径点中间作一个平滑查值后返回（可微分的）这个轨迹得到一个cost 加上任务级别的路径碰撞概率 作为整体的优化目标
##### LOSS 
障碍物cost 在欧几里德地图上计算  
与终点距离cost 在欧几里德地图上计算  
平滑cost 算每个路径点的经过时间
PLANNING NETWORK 还会预测一个路径碰撞概率 用二元交叉熵和真实碰撞概率对比 部署阶段只用这个概率<0.5的情况
这个仿真平台是用很多仿真图像和现实图像还原的

# ViPlanner: Visual Semantic Imperative Learning for Local Navigation
![](./navigation/viplanner.png)  
在Iplaner的基础上加入了语义图像 也是可微分的 一起训练 语义图像会被预处理   
Iplanner本质上训练用的是简化的立方体几何模型 这里使用了四足机器人 而且可以通行阶梯这种 （maybe因为语义信息） 但这几篇文章其实都没有考虑到机器人本身的动力学 就是作为一个移动agent 另外 和iplanner的区别是这里的运动用了RL模型 Iplanner是mpc viplanner可以在isaac lab上跑

# Learned Perceptive Forward Dynamics Model for Safe and Platform-aware Robotic Navigation
RSS 2025
考虑了机器人本体动力学　整体框架仍然是一个mpc的规划器　在给定机器人一个序列状态　和一个目标点的情况下规划出一条action　序列　（ＭＰＰＩ方式优化　reward设计）　　
这个序列状态　设计了一个动力学模型去预测　在已知当前和历史状态的情况下预测未来状态　这个模型要提前利用仿真数据和现实数据训练 也就是说 用一个已经训练好的rl locomotion 模型去跑 然后预测出一个模型来 可以根据已有观测预测未来机器人的位置 成功率什么的 这个预测出来的状态序列作为mppi规划的参数   
然后得到一系列点后 rl就可以跑了 部署 

# TOP-Nav: Legged Navigation Integrating Terrain,Obstacle and Proprioception Estimation
RSS 2025
下层用了rl的控制器 就是extreme parkour  
上层是一个计算代价的规划器 把视觉图像分块 计算障碍物成本 语义的地形可通过成本 以及下层控制器那个critic的运动成本也用上 把每一个patch的每一个成本都算出来 然后每规划一次都计算成本 然后随机采样点 选择最好的那个点 交给下层去做     
选出点后 交给rl去跑
## 感觉和上一篇一样 其实都是考虑了机器人的动力学的感觉 上一篇考虑了成功率 用了一个正常的x y w的速度跟踪模型 这一篇则考虑的是critic的运动值 给一个yaw拿给extreme parkour去跑