# Agile perceptive multiskill locomotion for quadrupedal robots in the wild

## 基本信息

- 作者：Jun-Gill Kang、Jaehyun Park、Tae-Gyu Song、Joon-Ha Kim、Seungwoo Hong、Hae-Won Park 等
- 发表信息：Science Robotics, 2026, 11, eadz7397；PDF 标注日期为 2026-07-15
- 原文：本地归档 [PDF](../原文/Agile%20perceptive%20multiskill%20locomotion%20for%20quadrupedal%20robots%20in%20the%20wild.pdf)
- 机器人平台：KAIST HOUND 四足机器人
- 主路线：[03 Locomotion](../../技术路线/03_Locomotion/README.md)
- 交叉路线：[01 足式运动基础与控制](../../技术路线/01_足式运动基础与控制/README.md)、[04 Sim2Real与实机部署](../../技术路线/04_Sim2Real与实机部署/README.md)
- 标签：`quadruped` `perceptive-locomotion` `multiskill` `motion-prior` `sim2real`

## 初步定位

论文提出 APT-RL（action pretrained transformer-based reinforcement learning），目标是在仅使用机载感知和计算资源的条件下，让四足机器人在复杂地形中进行高速、多技能运动，并自主完成步态和技能切换。

## 方法摘要

1. 使用简化的单刚体动力学和轨迹优化生成大规模二维运动数据，同时生成状态轨迹和对应控制输入。
2. 用 Transformer-based VAE 学习可复用的运动潜表示，并通过模仿学习获得 torque decoder。
3. 将预训练运动技能作为先验，继续学习复杂环境中的三维感知运动和技能转换。
4. 在 HOUND 上使用 LiDAR、深度相机和机载计算进行真实环境验证。

## 控制闭环初读

- 感知：机载 LiDAR、深度相机和本体状态。
- 中间表示：由预训练运动数据形成的潜在技能表示。
- 动作侧：方法强调直接学习包含控制输入的信息，避免只生成状态轨迹后再额外训练控制器。
- 任务侧：复杂地形通行、步态切换、多技能组合和高速障碍跨越。
- 实机证据：论文摘要报告了室内障碍和野外环境实验，并提到瞬时峰值速度最高约 6 m/s。

## 与本工程的价值

这篇论文适合作为“运动先验如何与感知 Locomotion 融合”的入口。它把数据生成、技能预训练、感知运动和 Sim2Real 放在同一条链路中，后续阅读应重点拆分每一阶段的输入输出，而不是只看最终策略效果。

## 后续完整分析待核对

- 预训练阶段和下游强化学习阶段的观测、动作、奖励是否完全一致。
- 二维轨迹如何扩展到三维地形和真实足端接触。
- Transformer/VAE 潜变量如何影响技能切换，是否存在显式选择器或完全由策略决定。
- 零样本 Sim2Real 的随机化、系统辨识、延迟和安全保护细节。
- 6 m/s 峰值速度对应的动作、地形和持续时间，以及与平均速度的区别。
- 是否公开代码、数据、机器人配置和可复现实验脚本。
