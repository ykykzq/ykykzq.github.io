---
title: 'ElegantRL项目学习-PPO算法'
date: 2025-04-1
permalink: /posts/2025/4/blog-post-1/
tags:
  - RL
  - PPO
  - ElegantRL
  - DL
---

开源强化学习项目[ElegantRL](https://github.com/AI4Finance-Foundation/ElegantRL)的代码阅读笔记。

## 1、PPO算法解析

PPO算法即近端策略优化算法，他基于策略梯度算法进行实现。

### 1.1 改进之处

原有的策略梯队（PG）算法存在问题，**在更新完θ之后**，原有的采样数据就不能再使用了，因此必须花非常多的时间进行采样。

PPO算法则**借鉴异策略算法的思想**，通过设置另外一个策略，并使用**重要性采样**的概率论理论，使用θ‘的训练数据来更新θ。

但是通过**KL散度约束**，可以使得θ'与θ的差距非常小，因此PPO本质上还是一个**同策略算法**。

## 2、源代码阅读分析

### 2.1 公有方法

#### 2.1.1 get_gym_env_args方法

get_gym_env_args是ElegantRL框架中的一个工具函数，用于从OpenAI Gym环境中提取关键信息，并将其组织为标准化的字典格式。

方法返回的类型是一个**dict**，从env中获取

- env可以是标准gym环境，也可以是自定义的环境，如果是标准环境，则处理状态空间与动作空间（主要是检测维度）
- 如果是自定义环境，则直接从环境对象的属性中获取必要信息。这要求自定义环境实现这些属性。

最后，将所有信息整合成标准字典格式。

#### 2.1.2 kwargs_filter方法

kwargs_filter方法用于从传入的参数字典中筛选出函数能够接受的参数。

他接收一个func，一个dict，返回的是func能接收的dict。

#### 2.1.3 build_env方法

build_env方法用于构建强化学习环境实例。

这个函数在训练和评估过程中都会被调用，它负责将环境类和环境参数转换为可交互的环境对象。

接收一个类，一个参数列表，然后基于该参数列表，实例化该类。

- 方法中调用了kwargs_filter方法。

#### 2.1.4 build_mlp方法

用于构建多层感知机神经网络。他接收一个神经网络各层的维度列表，包括输入层、隐藏层和输出层的维度，然后输出一个nn.Sequential。

具体的构建过程为逐步一层层添加线性层，并在添加完之后删除激活函数。

#### 2.1.5 layer_init_with_orthogonal方法

layer_init_with_orthogonal是ElegantRL框架中用于**初始化神经网络层权重**的工具函数。具体的步骤为**权重正交初始化**和**偏置项常数初始化**，方法是基于std与bias_const进行初始化。

#### 2.1.6 get_rewards_and_steps方法

执行单次评估。

#### 2.1.7 draw_learning_curve_using_recorder方法

绘制学习曲线用。

#### 2.1.8 train_agent方法

会调用Evaluator类的**evaluate_and_save**方法，持续交互，直到达到设定的break_step。

#### 2.1.9 valid_agent方法

验证已训练好的agent的性能。加载训练好参数并进行评估。

#### 2.1.10 train_ppo_for_pendulum和train_ppo_for_lunar_lander

两个训练过程，调用**train_agent**方法来进行两个不同game的训练。

### 2.2 Config类

Config类是ElegantRL框架中的核心配置类，用于保存和管理强化学习算法训练过程中的所有超参数和设置。他有三个方法，分别是用于初始化的\__init__方法、用于在训练前初始化的init_before_training方法和用于判断是否为离策略算法的get_if_off_policy方法。

**\__init__**方法负责初始化一系列参数，主要包括**奖励塑形参数**（如未来奖励的折扣因子）、**训练相关参数**（如网络的维度和学习率）、**设备相关参数**（如使用的GPU）和**评估相关参数**（如每多少steps评估一次）。

**init_before_training**方法负责设置训练时的随机种子，并完成工作目录（保存log等）的设置。

**get_if_off_policy**方法根据算法的名称来判断算法为**off_policy**方法（如DQN、Q-Learning）还是**on_policy**方法（如TRPO、A2C、A3C）。

### 2.3 ActorPPO类

ActorPPO类是PPO算法中负责**生成动作**的策略网络类。

#### 2.3.1 初始化过程

- 网络结构为mlp，维度结构为[状态维度，网络维度，动作维度]。
- 可训练参数action_std_log：动作标准差的对数值，作为可训练参数。
- 指定概率分布ActionDist：在连续动作时使用，初始化为正态分布。

#### 2.3.2 forward方法——前向传播过程

将状态传入到net中，然后再转换动作到环境可接受的范围。

- 转换过程是通过调用convert_action_for_env方法使用的，在该转换过程中，使用tanh将动作压缩到[-1,1]区间。

#### 2.3.3 get_action方法

get_action在训练阶段生成随机动作以进行环境探索。

返回值为Tuple[TEN, TEN]，为**采样的动作**和**动作对应的对数概率**。

计算的过程是先通过net获得动作的平均值，再根据初始化时指定的action_std_log参数和概率分布来设置分布，再从分布中获得动作。

区别：

- get_action在训练时探索，前向传播（forward方法）过程用于推理。
- get_action引入了随机性，forward方法是确定的，不存在随机性。

#### 2.3.4 get_logprob_entropy方法

用于计算给定状态和动作对的对数概率以及策略的熵值。输入参数为**当前状态向量**和**已执行的动作向量**。

返回一个张量列表，为[给定动作在当前策略下的对数概率, 当前策略的熵值]

### 2.4 CriticPPO类

使用一个mlp进行**当前状态的价值估计**。

价值可以理解为从该状态开始，按照当前策略行动，预期能够获得的未来折扣奖励总和。

### 2.5 AgentPPO类

AgentPPO类负责协调策略网络(Actor)和价值网络(Critic)的训练、与环境的交互以及更新策略。

#### 2.5.1 初始化过程

- 决定是否为**连续动作空间**和
- 网络维度、状态维度、动作维度的初始化
- 设置训练参数，例如batch_size、gamma、repeat_times、reward_scale、学习率等
- 创建actor与critic网络
- 设置ratio_clip等PPO特有的参数

#### 2.5.2 explore_env方法

explore_env方法负责智能体与环境交互并收集训练数据。他接收的参数包括环境、交互的步数和其他一些关键字参数。

其返回的值包括：

- **states**: 状态张量，形状为(horizon_len, state_dim)
- **actions**: 动作张量，形状为(horizon_len, action_dim)
- **logprobs**: 动作的对数概率，形状为(horizon_len,)
- **rewards**: 奖励，形状为(horizon_len, 1)
- **undones**: 非终止标志，形状为(horizon_len, 1)
- **unmasks**: 非截断标志，形状为(horizon_len, 1)

其执行的过程主要是先初始化，再进行n次交互，收集每一次交互的数据并保存在缓冲区中。

- 如果是**截断**导致的状态结束，使用价值网络估计的未来奖励作为补偿，减少由于人为截断导致的价值估计偏差。

- 在这个过程中，会调用**explore_action**方法

#### 2.5.3 explore_action方法

explore_action方法负责对单个状态生成随机动作，用于智能体与环境的交互和探索。

- 在该过程中，调用actor网络的get_action方法。

#### 2.5.4 update_net方法

update_net方法进行PPO算法的**策略更新**。

其返回值的含义为：

- **obj_critic_avg**: 价值网络损失的平均值
- **obj_actor_avg**: 策略网络目标函数的平均值
- **a_std_log.item()**: 动作标准差对数的平均值(用于监控探索程度)

其执行过程为：

1. 首先解包输入的buffer元组，获取所有数据，并记录数据量
2. 使用价值网络(Critic)对所有状态进行评估，获取values
3. 计算广义优势估计(GAE)，在此过程中会调用**get_advantages**方法获取GAE
4. 对优势值进行标准化处理，并构建更新所需要的数据
5. 多次使用同一批数据更新网络，在此过程中会调用**update_objectives**方法。

#### 2.5.5 update_objectives方法

update_objectives方法负责执行**单次**的**策略**和**价值**网络的参数更新。

1. 随机采样批次数据
2. 更新价值网络(Critic)。具体来说，使用平滑L1损失(SmoothL1Loss)计算预测值与目标值之间的差异、乘以unmask将截断状态的损失排除(这些状态的回报估计不完整)、调用**optimizer_backward**方法执行梯度下降，更新价值网络参数。
3. 更新策略网络(Actor)。这是最关键的部分。
   1. 使用**get_logprob_entropy**方法，获取在当前策略下，给定动作的对数概率和熵。
   2. 计算**标准目标函数**和计算**截断目标函数**，取二者的最小值，实现策略更新。
   3. 添加熵正则化以防止过早收敛到次优解。
   4. 进行梯度上升。

#### 2.5.6 get_advantages方法

get_advantages方法负责计算广义优势估计(Generalized Advantage Estimation, GAE)。

<font color =red>待学习完GAE后补充</font >。

#### 2.5.7 optimizer_backward方法

是一个辅助方法，封装了优化器的标准更新步骤：

1. 清除之前的梯度
2. 计算目标函数的梯度
3. 执行一步参数更新

### 2.6 Evaluator类

Evaluator类是ElegantRL框架中的一个关键组件，负责评估智能体的性能、保存模型和记录训练进度。

#### 2.6.1 初始化过程

设置用于评估的环境实例、评估频率（几个step评估一次）、每次评估运行的次数（多次运行以计算**平均**回报）和当前工作目录。

#### 2.6.2 evaluate_and_save方法

判断是否需要评估、运行多次评估、计算统计数据、记录训练数据、保存模型。

#### 2.6.3 close方法

训练结束时调用，完成两项工作：

1. 保存记录：将recorder列表保存为numpy数组
2. 绘制学习曲线：调用**draw_learning_curve_using_recorder**函数绘制训练曲线



