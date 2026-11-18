# SuperMimic: Scaling Humanoid Whole-Body Control with One Differentiable World Model

## 基本信息

- 作者：Qiayuan Liao、Yiyang Shao、Junfeng Long、Kevin Zakka、Roberto Horowitz、Pieter Abbeel、Koushil Sreenath
- 来源：arXiv:2610.28480v1，2026-10-20；CoRL 2026
- 原文：本地归档 [PDF](../原文/SuperMimic%20Scaling%20Humanoid%20Whole-Body%20Control%20with%20One%20Differentiable%20World%20Model.pdf)
- 项目页：<https://supermimic.github.io/>
- 机器人平台：Unitree G1 人形机器人；仿真使用 MuJoCo Warp
- 主路线：[03 Locomotion](../../技术路线/03_Locomotion/README.md)
- 交叉路线：[01 足式运动基础与控制](../../技术路线/01_足式运动基础与控制/README.md)、[04 Sim2Real与实机部署](../../技术路线/04_Sim2Real与实机部署/README.md)、[06 世界模型与预测控制](../../技术路线/06_世界模型与预测控制/README.md)、[08 数据平台](../../技术路线/08_数据平台/README.md)
- 标签：`humanoid` `whole-body-control` `motion-tracking` `world-model` `model-based-rl` `sim2real`

## 初步定位

SuperMimic 面向大规模人形机器人全身动作跟踪。它不是为每个动作训练专家策略，也不是用大规模 PPO 直接堆算力，而是用一个从刚体仿真器 transition 数据中学习到的可微世界模型表示共享的人形动力学，再通过短时想象 rollout 的 BPTT 给策略提供解析梯度。

这篇论文对本工程的主要价值在于：把“动作库规模化”“世界模型”“全身控制”和“真机迁移”放在同一条训练链路里，提供了一个比传统 model-free RL 更省算力的足式全身控制训练范式。

## 最新成果

截至 2026-10-24，论文和项目页给出的最新结果包括：

1. 计算效率：在 1 张 H100 上训练 3 小时即超过 SONIC 发布 checkpoint 的 normalized return。SONIC 对比项为 128 张 L40 训练 7 天，约 21,000 GPU-hours；论文保守表述为墙钟时间减少超过 50 倍、GPU-hours 减少超过 1000 倍。
2. 动作库覆盖：在过滤后的 BONES-SEED G1 动作集上评测 129,514 个动作，SuperMimic 在 1 张 H100 训练 3 小时后失败 25 个动作，2 张 H100 训练 9.8 小时后失败数为 0；SONIC 发布 checkpoint 失败 179 个动作。
3. 扩展性：当动作数据从 1 小时扩展到 264 小时时，世界模型的 8-step prediction error 保持在 0.007 到 0.011，达到目标 return 所需 GPU-hours 增长较慢。
4. 实机迁移：同一个通用 motion tracker 不针对单个动作调参即可迁移到真实 Unitree G1，展示了 skipping jump、side roll、90 度 flip 和原地 360 度 flip。
5. 下游接口：初步速度命令接口实验中，使用世界模型和速度跟踪奖励训练的 RL encoder 在 672 个 step response 中有 650 个稳态误差低于蒸馏 baseline；蒸馏 baseline 在 7 个快速转向命令上全失败。

## 方法摘要

1. 当前策略在 MuJoCo Warp 中收集 transition，并写入固定大小 replay buffer。
2. 可微世界模型从 replay buffer 中采样 k-step 窗口训练，预测下一状态、奖励、观测和终止概率。
3. 策略更新时冻结世界模型，通过 H=32 的 imagined rollout 反向传播 return，并用 critic bootstrap 想象 horizon 之后的价值。
4. motion-tracking goal 使用未来 K=10 帧参考动作序列；其他 goal interface 可以替换为速度命令、末端位姿或 SMPL body-pose 等形式。
5. 动作是归一化关节 setpoint，低 PD gain 使其更接近关节力矩代理；部署 actor 使用带噪本体观测和 goal 输入，不直接依赖仿真器梯度。

## 控制闭环初读

- 观测：关节位置相对默认姿态、关节速度、IMU 基座角速度、基座系重力投影、上一时刻动作；策略使用最近 10 帧观测历史。
- 目标：默认是全参考动作跟踪，使用未来 10 帧 reference；下游速度接口使用 `[vx, vy, wz]` 命令和观测历史特征。
- 动作：输出归一化关节 setpoint `q_sp = alpha * a + q0`，由低增益 PD 执行。
- 世界模型上下文：摩擦系数、关节标定偏移、躯干质心偏移等 domain-randomization 参数。
- 训练频率：动作与世界模型 step 以 50 Hz 控制步为单位；MuJoCo Warp 每个控制步使用 4 个 5 ms substep。
- 真机证据：Unitree G1 上展示多种全身高动态动作，但论文没有把外部地形感知或物体交互纳入当前系统。

## 优势分析

- 算力优势明确：核心增益来自把共享身体动力学学习成一个可微世界模型，使新增动作主要增加对共享动力学的证据，而不是线性增加 model-free rollout 成本。
- 梯度更高效：BPTT 通过世界模型直接得到动作对 return 的一阶梯度；论文报告相对于在世界模型 rollout 上跑 PPO，达到相同 best return 约少 64 倍 imagined steps。
- rollout 更快：在实际使用的 batch size 范围内，世界模型 rollout 比 MuJoCo Warp 快 10 倍以上，大 batch 下最高报告约 62 倍。
- 不要求可微仿真器：它不反向传播 MuJoCo 本体，而是用标准刚体仿真器日志拟合可微替代模型，因此原则上更容易接入现有非光滑接触仿真器。
- 泛化到接口训练：同一个世界模型可以训练不同 goal encoder，初步速度命令实验表明，直接用任务奖励训练接口可能优于仅从 reference motion 蒸馏。
- 对足式机器人有迁移启发：虽然论文平台是人形 Unitree G1，不是四足，但“动作库 + 世界模型 + 短 horizon BPTT + Sim2Real”的训练结构可借鉴到双足、四足和轮足的高动态技能库训练。

## 与本工程的价值

这篇论文应作为 [03 Locomotion](../../技术路线/03_Locomotion/README.md) 中“全身动作跟踪和技能库规模化”的重点条目，同时在 [06 世界模型与预测控制](../../技术路线/06_世界模型与预测控制/README.md) 中作为“世界模型真正进入动作选择和策略梯度”的代表案例。它也补充了 [04 Sim2Real与实机部署](../../技术路线/04_Sim2Real与实机部署/README.md) 对域随机化、执行器接口和高动态人形实机验证的关注点。

## 复现边界

- 官方项目页当前提供论文、视频和交互 demo 入口；未在项目页看到官方代码仓库入口。
- 训练数据使用公开 G1-retargeted BONES-SEED，并经过名称过滤和可行性过滤后保留 129,514 clips；速度接口实验另使用 LAFAN1 和 BONES-SEED 的 walking/running 子集。
- 论文报告的主要训练资源是 NVIDIA H100 80GB SXM5；对比 SONIC 时硬件、数据量和实现细节不同，需按论文的 compute accounting 理解，而不能直接等价为同平台 benchmark。
- 评测的 129,514 个动作是过滤后训练集 benchmark，主要衡量训练动作覆盖，不等同于新动作泛化。

## 局限与后续问题

- 世界模型只从仿真 transition 学习，尚未用真实机器人数据做模型修正或系统辨识。
- 当前只训练平地动作，不包含地形或物体交互；对本工程的复杂地形 Locomotion 还需要额外感知和接触建模。
- 多接口训练仍是初步研究：论文的速度接口奖励只更新 velocity encoder，不更新共享 decoder；接口联合训练是否真正必要仍未验证。
- 对四足或轮足平台的迁移价值需要重新定义动作库、goal interface、关节 setpoint/力矩接口和接触终止条件。
- 需要继续跟踪是否发布训练代码、MuJoCo Warp 环境、BONES-SEED 过滤脚本和 Unitree G1 部署配置。
