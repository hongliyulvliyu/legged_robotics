# 07 仿真平台

本专题用于整理足式机器人相关仿真平台、训练环境和关键使用技巧。它是横向支撑专题，不替代具体算法路线；涉及 Locomotion、Sim2Real 或导航的论文仍归入对应技术路线。

## 平台范围

| 平台 | 关注点 |
| --- | --- |
| Isaac / Isaac Lab / Isaac Sim | GPU 并行训练、传感器仿真、资产导入、场景随机化和策略部署接口 |
| MuJoCo | 轻量动力学仿真、模型调试、控制器验证和 Sim2Sim 对照 |
| 宇树仿真器与 Unitree 工具链 | Unitree Go2、H1、G1 等平台的训练、仿真、Sim2Sim 和实机接口 |
| Genesis / Raisim / Webots 等 | 作为对照仿真器或专项任务环境，根据实际使用经验补充 |

## 使用技巧

| 平台 | 主题 | 链接 | 简洁记录 |
| --- | --- | --- | --- |
| Isaac Sim | PhysX Lidar 与 128 线雷达对齐 | [笔记](Isaac_Sim_PhysX_Lidar.md) · [微信原文](https://mp.weixin.qq.com/s/biKBVUyNkxG1Zt8GsuDbTA) | 记录 Isaac Sim 中 PhysX SDK Lidar 的 GUI 示例、RS-Ruby Plus 128 线参数等效、Python 取点云和常见坑。 |

## 记录模板

```markdown
## 平台/工具名称

- 适用机器人：
- 适用任务：训练 / 推理 / 传感器仿真 / Sim2Sim / 部署前验证
- 关键版本：
- 依赖环境：
- 模型格式：URDF / MJCF / USD / 其他
- 传感器支持：
- 与实机接口：
- 常见坑：
- 参考链接：
```

## 与其他路线的关系

- 训练环境、奖励和策略接口可回链到 [01 足式运动基础与控制](../01_足式运动基础与控制/README.md)。
- 传感器、地图和导航接口可回链到 [02 基础感知与导航](../02_基础感知与导航/README.md)。
- 复杂地形和感知运动任务可回链到 [03 Locomotion](../03_Locomotion/README.md)。
- 参数辨识、Sim2Sim 和真机迁移可回链到 [04 Sim2Real与实机部署](../04_Sim2Real与实机部署/README.md)。
