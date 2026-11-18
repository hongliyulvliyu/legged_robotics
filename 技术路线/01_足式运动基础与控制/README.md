# 01 足式运动基础与控制

本路线负责足式机器人本体运动的基础知识和控制基座。

## 内容范围

- 运动学、动力学、浮基模型和接触建模
- 足端、质心、姿态和全身控制
- MPC、WBC、QP、阻抗控制和状态反馈
- PPO、策略观测、动作接口、奖励和课程学习
- 运动先验、模仿学习和技能表示的控制侧基础

## 阅读重点

优先确认模型假设、接触处理、观测与动作定义、控制频率、训练特权信息，以及策略是否真正承担了低层控制。

## 相关论文

- [APT-RL：Agile perceptive multiskill locomotion](../../论文/初步阅读/Agile_perceptive_multiskill_locomotion.md)：以轨迹优化生成的状态与控制数据预训练运动技能，再作为强化学习的运动先验。
- [SuperMimic](../../论文/初步阅读/SuperMimic.md)：关注人形全身 motion tracking 的观测、动作、奖励、低增益 PD setpoint 接口，以及 BPTT 通过可微世界模型更新控制策略。

## 交叉路线

- 感知参与闭环时进入 [03 Locomotion](../03_Locomotion/README.md)。
- 参数偏差、迁移和真机安全进入 [04 Sim2Real与实机部署](../04_Sim2Real与实机部署/README.md)。
