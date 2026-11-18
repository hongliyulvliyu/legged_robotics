# 03 Locomotion

本路线专门维护感知与运动控制融合的足式 Locomotion 问题。

## 内容范围

- 感知驱动的步态、落足点和运动模式切换
- 复杂地形、障碍跨越、跑酷和野外运动
- 视觉、激光、触觉与本体历史共同参与的闭环控制
- 多技能、多步态和技能过渡
- 从导航命令到低层足式运动的端到端或分层系统

## 阅读重点

重点记录地形输入如何改变控制决策、技能如何切换、失败如何恢复，以及感知策略和低层控制之间的接口。

当前归档的论文均与本路线相关：APT-RL偏向多技能感知运动，SEA-Nav偏向导航策略与低层 Locomotion 控制器的安全接口，SuperMimic偏向人形全身动作跟踪和技能库规模化训练。

## 相关论文

| 论文 | 关注点 |
| --- | --- |
| [APT-RL：Agile perceptive multiskill locomotion](../../论文/初步阅读/Agile_perceptive_multiskill_locomotion.md) | 运动先验、多技能表示、步态切换和复杂地形感知运动 |
| [SEA-Nav](../../论文/初步阅读/SEA-Nav.md) | 高层导航策略、速度命令、安全投影和低层 Locomotion 接口 |
| [SuperMimic](../../论文/初步阅读/SuperMimic.md) | 人形全身 motion tracking、动作库规模化、世界模型加速策略训练和 Unitree G1 实机迁移 |
