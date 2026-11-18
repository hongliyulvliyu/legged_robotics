# UniLab 工程概览

GitHub：[unilabsim/UniLab](https://github.com/unilabsim/UniLab)

项目页：[UniLab: A Heterogeneous Architecture for Robot RL](https://unilabsim.github.io/)

文档：[UniLab Documentation](https://unilabsim.github.io/UniLab-doc/en/0-index.html)

## 一句话定位

UniLab 是一个面向机器人强化学习的可配置训练基础设施。它把任务语义、物理后端和学习器运行时拆开：任务通过 Hydra 和 manager terms 组合，物理仿真通过统一 `SimBackend` contract 接入，训练和评估通过同一套 CLI 流程完成。

它更像是机器人 RL 的“训练与任务组织层”，而不是单一仿真器；核心价值在于让同一套任务定义可以尽量复用到不同物理后端、不同硬件平台和不同 RL 算法上。

## 关键特点

| 方向 | 说明 |
| --- | --- |
| 任务配置化 | 动作、观测、奖励、终止条件、事件、命令、课程学习和指标等都以 manager term 形式组织在 Hydra YAML 中；已有组件的变体通常不需要新写环境类。 |
| 后端可切换 | 物理后端遵循统一 `SimBackend` contract，可通过 `--sim` 选择，例如 MuJoCo 或 Motrix；任务侧流程尽量保持一致，后端差异显式留在配置中。 |
| 异构训练架构 | CPU 侧并行仿真或外部 worker 可以向加速器学习器供数，仿真器不必先变成 CUDA 驻留仿真器；学习侧可覆盖 CUDA、ROCm、MPS、XPU 等平台。 |
| 采样与更新重叠 | SAC / FlashSAC 等 replay-based off-policy 路径支持数据采集和学习器更新重叠，项目页和 README 报告了代表配置上的端到端加速收益。 |
| 统一 CLI | 训练、评估、回放和恢复通过可读的命令参数暴露算法、任务和仿真器选择，例如 `uv run train --algo ppo --task go2_joystick_flat --sim mujoco`。 |
| 生态化定位 | UniLab 设计为机器人专用仓库的共享任务和训练表面，下游仓库可以独立维护机器人 recipe，同时复用任务、后端和 RL contract。 |

## 主要优势

| 优势 | 对足式机器人研发的意义 |
| --- | --- |
| 降低任务迁移成本 | Locomotion、motion tracking、loco-manip 等任务如果能复用同一套观测、奖励和终止项，后续切换后端或机器人型号时改动更集中。 |
| 更适合混合硬件环境 | 不强制所有训练都绑定 Linux + CUDA + GPU resident simulator；对 Apple Silicon、ROCm、Intel XPU 或 CPU 仿真资源更友好，便于在不同机器上做开发、冒烟和小规模实验。 |
| 便于 Sim2Sim 对照 | 同一任务可以在 MuJoCo / Motrix 等后端之间切换，用于排查策略是否过度依赖某个物理引擎的细节。 |
| 端到端吞吐思路清晰 | 把 CPU rollout、共享内存 IPC、异步权重同步和 GPU learner 分开调度，适合分析训练瓶颈，不只是盲目增加并行环境数。 |
| 算法覆盖较宽 | 项目页列出 PPO、APPO、SAC、TD3、FlashSAC、HORA、HIM-PPO 等算法路径，便于在 on-policy、off-policy 和蒸馏路线之间对比。 |
| 机器人类别覆盖广 | 项目页覆盖机械臂、四足、人形、灵巧手和轮足等类别，任务族包括行走、跑酷、舞蹈、翻转、滑行、移动操作和灵巧操作。 |

## 适用场景

| 场景 | 推荐理由 |
| --- | --- |
| 四足 / 人形 RL 训练框架调研 | UniLab 的任务配置、后端 contract 和异构 runtime 代表了不同于 Isaac Lab / Isaac Gym GPU 驻留路线的系统设计。 |
| 需要跨仿真器验证策略 | 可把 UniLab 作为 MuJoCo、Motrix 等后端之间的统一任务入口，观察奖励、观测和控制接口在 Sim2Sim 中的稳定性。 |
| 非 CUDA 环境下做训练或开发 | 对 macOS、ROCm、XPU 等平台有文档化路径，适合做轻量开发、算法验证或没有 NVIDIA 工作站时的实验入口。 |
| 研究 off-policy 训练加速 | FlashSAC / SAC 的异步采样与更新路径适合关注 replay-based 机器人 RL 吞吐优化的场景。 |
| 机器人专用 recipe 复用 | 下游机器人仓库可以独立维护资产、任务和参数，同时沿用 UniLab 的任务、后端和学习器 contract。 |

## 与其他仿真平台的关系

| 平台 | 与 UniLab 的关系 |
| --- | --- |
| Isaac Lab | Isaac Lab 更偏 GPU 驻留物理和成熟 manager-based 任务 API；UniLab 借鉴 manager 式任务组织，但强调 CPU 仿真、加速器学习和跨硬件运行时解耦。 |
| MuJoCo | MuJoCo 可作为 UniLab 的后端之一，用于轻量动力学训练、调试和 Sim2Sim 对照。 |
| MotrixSim | UniLab 项目页把 MotrixSim 与 MuJoCo 一起列为主要物理后端，用于批量 CPU 刚体仿真。 |
| Genesis / Raisim / Webots | 可作为对照仿真器关注；若未来 UniLab 后端 contract 覆盖这些平台，可评估是否纳入统一任务入口。 |

## 使用入口

```bash
git clone https://github.com/unilabsim/UniLab.git
cd UniLab
make setup
uv run demo dance
```

训练与评估示例：

```bash
uv run train --algo ppo --task go2_joystick_flat --sim motrix
uv run eval --algo ppo --task go2_joystick_flat --sim motrix --load-run -1
uv run train --algo ppo --task go2_joystick_flat --sim mujoco
uv run train --algo sac --task g1_walk_flat --sim mujoco
```

## 注意事项

- UniLab 不应简单归类为“又一个物理引擎”；评估时要分清任务抽象层、物理后端、RL runtime 和机器人 recipe 各自的边界。
- 后端和任务成熟度需要看官方 support matrix；不要默认所有机器人、算法和物理后端组合都已完整验证。
- 如果目标是高保真传感器仿真、RTX 渲染或复杂视觉数据闭环，Isaac Sim / Isaac Lab 仍然是更直接的起点；UniLab 更适合关注 RL 训练管线、任务复用和后端切换。
- 如果目标是实机部署，需要额外检查导出的策略接口、控制频率、观测归一化、动作缩放、延迟和安全约束，本条目只覆盖仿真训练平台层面的信息。

## 参考链接

- GitHub README：https://github.com/unilabsim/UniLab
- 项目页：https://unilabsim.github.io/
- 官方文档：https://unilabsim.github.io/UniLab-doc/en/0-index.html
- 论文：https://arxiv.org/abs/2605.30313
