# SEA-Nav: Efficient Policy Learning for Safe and Agile Quadruped Navigation in Cluttered Environments

## 基本信息

- 作者：Shiyi Chen、Mingye Yang、Haiyan Mao、Jiaqi Zhang、Haiyi Liu、Shuheng He、Debing Zhang、Zihao Qiu、Chun Zhang 等
- 来源：arXiv:2603.09460v1，2026-03-10
- 原文：本地归档 [PDF](../原文/SEA-Nav.pdf)
- 机器人平台：四足机器人；论文摘要强调真实世界部署
- 主路线：[02 基础感知与导航](../../技术路线/02_基础感知与导航/README.md)
- 交叉路线：[03 Locomotion](../../技术路线/03_Locomotion/README.md)、[04 Sim2Real与实机部署](../../技术路线/04_Sim2Real与实机部署/README.md)
- 标签：`quadruped` `navigation` `safe-rl` `lidar` `cbf` `sim2real`

## 初步定位

SEA-Nav 研究密集障碍环境中的四足导航，核心目标是在训练效率、安全性和运动敏捷性之间取得平衡。方法将强化学习导航策略、可微分控制屏障函数和针对高风险状态的经验重放结合起来。

## 方法摘要

1. 使用 Adaptive Collision-State Initialization（ACSI）在碰撞前的高风险状态附近重置，增加关键避障经验的采样概率。
2. 使用自适应 LSE-CBF 安全层，把多个 LiDAR 约束融合为可微的安全速度投影，并通过物理阻尼减少狭窄通道中的振荡。
3. 策略同时输出名义速度命令和安全增益，安全层计算最终速度命令。
4. 通过运动学动作约束和正则化，抑制真实部署中危险的高速转向。

## 控制闭环初读

- 外感知：二维 LiDAR 和局部目标点；论文中外感知频率为 10 Hz。
- 本体状态：基座线速度、角速度和重力投影等本体观测；本体侧频率为 50 Hz。
- 策略输出：三维机体速度命令 `[vx, vy, wz]` 以及自适应安全增益。
- 低层执行：安全速度命令发送给预训练的低层 Locomotion 控制器，再转换为关节力矩。
- 训练：PPO 与安全层、碰撞状态重放和运动学正则化联合使用。
- 部署：论文摘要声称在单张 RTX 4090 上经过分钟级训练，并在未见迷宫中进行零样本真实部署。

## 与本工程的价值

这篇论文适合作为“导航策略如何通过明确接口调用足式 Locomotion”的入口。它的主要贡献不在重新学习低层步态，而在将安全约束、经验采样和高层速度命令接入已有低层运动控制器。

## 后续完整分析待核对

- CBF 安全层的离散时间安全性质、适用假设和实际碰撞边界。
- LSE 融合、阻尼项和自适应增益对狭窄通道振荡的具体影响。
- ACSI 对训练速度的增益是否来自状态分布改变，及其与普通课程学习的公平对比。
- 预训练低层 Locomotion 控制器的平台、观测、动作和 Sim2Real 过程。
- 论文所称“安全”覆盖的是速度命令约束、几何碰撞还是整机失稳风险。
- 真实迷宫的传感器、地图、目标点来源、失败案例和可复现实验条件。
