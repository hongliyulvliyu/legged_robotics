# Legged Robotics

本工程聚焦足式机器人，同时维护与足式机器人相关的通用 VLA、世界模型和预测控制内容。

本项目参考了原工程 [humanoid-motion-intelligence](https://github.com/RealXiaoze/humanoid-motion-intelligence) 的组织方式，尤其参考了“技术路线、论文与项目”的目录结构。这里只参考工程结构和维护方法，不直接搬运原工程中的论文、项目或分析内容。

原工程地址：<https://github.com/RealXiaoze/humanoid-motion-intelligence>

## 工程边界

- 以四足、双足、轮足和其他通用腿式机器人为主要对象。
- 与足式运动直接相关的感知、导航、Locomotion、Sim2Real 和实机部署内容优先收录。
- VLA、世界模型和预测控制作为相对不固定本体的新技术方向单独维护，但必须记录其与足式机器人的接口关系。
- 论文和项目按本工程的六条技术路线组织，不复制原工程已有条目。

## 目录

| 目录 | 内容 |
| --- | --- |
| [技术路线](技术路线/README.md) | 六条技术路线及其学习、研究和归档边界 |
| [论文](论文/README.md) | 原文归档、初步阅读和后续论文分析入口 |
| [项目](项目/README.md) | 开源项目、工程位置、复现边界和平台信息 |

## 当前论文

当前先完成本地 PDF 的归档和初步阅读，不展开完整论文解读：

- [Agile perceptive multiskill locomotion for quadrupedal robots in the wild](论文/初步阅读/Agile_perceptive_multiskill_locomotion.md)
- [SEA-Nav: Efficient Policy Learning for Safe and Agile Quadruped Navigation in Cluttered Environments](论文/初步阅读/SEA-Nav.md)
- [SuperMimic: Scaling Humanoid Whole-Body Control with One Differentiable World Model](论文/初步阅读/SuperMimic.md)

## 维护原则

每篇论文和每个项目都应明确：研究问题、机器人平台、观测与动作接口、所属路线、可复现边界，以及与足式机器人的直接关系。初步阅读只记录定位和后续问题，完整分析在后续单独补充。
