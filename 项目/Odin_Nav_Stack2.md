# Odin Navigation Stack 2 (ONS2) —— 留形科技 & 璇玑动力

GitHub：[ManifoldTechLtd/Odin-Nav-Stack2](https://github.com/ManifoldTechLtd/Odin-Nav-Stack2)

架构文档：[ONS2_ARCHITECTURE.md](https://github.com/ManifoldTechLtd/Odin-Nav-Stack2/blob/main/docs/architecture/ONS2_ARCHITECTURE.md) · [MODULE_DATAFLOW.md](https://github.com/ManifoldTechLtd/Odin-Nav-Stack2/blob/main/docs/architecture/MODULE_DATAFLOW.md)

## 基本信息

- 维护方：Manifold Tech / ASTRALL Dynamics
- 运行时：ROS1 Noetic（`ros_ws/`）与 ROS2 Jazzy（`ros2_ws/`）双栈，话题/服务/动作名对齐
- 机器人平台：ASTRALL SW01、Unitree Go2、Dobot Lumo；可通过 `robots/<robot>/robot.yaml` 接入自定义底盘
- 主路线：[02 基础感知与导航](../技术路线/02_基础感知与导航/README.md)
- 工程主表：[项目/README.md](README.md)
- 交叉路线：[03 Locomotion](../技术路线/03_Locomotion/README.md)（速度命令交给低层运控）、[06 世界模型与预测控制](../技术路线/06_世界模型与预测控制/README.md)（NeuPAN 作为 MPC 实例）
- 标签：`navigation` `odin1` `sliding_map` `A*` `NeuPAN` `SRU` `mission` `ros1` `ros2`

## 一句话定位

ONS2 是基于 Odin1 空间感知模块的地面平台巡检导航栈：在重定位与静态地图之上，用盲区障碍记忆、全局 A\* 与可插拔局部规划器（默认 NeuPAN，可选 SRU）完成多航点自主巡逻，并由任务状态机统一交接 `/cmd_vel` 控制权。

---

## 系统整体框架（分层总览）

```text
Odin1 定位/重定位（闭源）
    ↓ 点云 + 里程计 + TF
感知与 360° 障碍记忆（pc_to_scan → sliding_map）
    ↓ /scan_360、/sliding_map、/map
全局规划 map_planner（2D A* + LOS 剪枝 + Laplacian 平滑）
    ↓ /initial_path
局部规划：NeuPAN（默认）或 SRU（可选）
    ↓ /cmd_vel（互斥）
底盘桥（unitree / sw01 / dobot …）
        ↑
任务调度 mission_manager + 精对齐 atomic_move
```

| 层 | 模块 | 默认算法 / 做法 | 主要交互 |
| --- | --- | --- | --- |
| 定位 | `odin_ros_driver` | Odin1 内部 3D SLAM / 重定位（`custom_map_mode=2`） | `/odin1/cloud_slam`、`/odin1/odometry`、TF `map→odom→odin1_base_link` |
| 障碍记忆 | `pc_to_scan` + `sliding_map` | 点云高度切片 + 机体系滚动栅格盲区记忆 | `/scan` → `/sliding_map`、`/scan_360` |
| 全局规划 | `map_planner` | 8 连通 2D A\* + LOS shortcut + Laplacian | 服务 `/map_planner/plan` → `/initial_path` |
| 局部规划 | `neupan_ros` / SRU | NeuPAN：MPC+DUNE+NRMP；SRU：深度端到端策略 | Action `/neupan/nav_path` 或 `/sru/goal_pose` → `/cmd_vel` |
| 任务与对齐 | `mission_manager`、`atomic_move` | 多航点状态机；闭环 move/turn/align | 到点发 stop，交接 `/cmd_vel` 做最终朝向 |
| 执行 | 底盘 bridge | 订 `/cmd_vel` → SDK/RPC | 任意时刻最多一个节点发速度 |

插件槽（`robots/<robot>/robot.yaml`）：`global_planner`、`local_planner`、`bridge` 可替换，契约话题名可配置。

---

## Q1. sliding_map 盲区记忆

### 方案

**机体系滚动 OccupancyGrid + 射线写入 + 时间衰减**，用来补偿上游前向约 90° FOV（由 `pc_to_scan` 的 `angle_min/max≈±0.7853` 决定），不是永久全局地图。

| 项 | 实现 |
| --- | --- |
| 数据结构 | `nav_msgs/OccupancyGrid`：`-1` unknown / `0` free / `100` occupied |
| 尺寸 | 默认 `resolution=0.05` m，`width=height=400` → 约 **20×20 m** |
| 坐标系 | 常用 `map_frame=odom`；偏离中心超过 `recenter_threshold_m`（默认约 5 m）时平移栅格保留数据 |

### 占用如何写入

1. **自由射线（ray clearing）**：沿每根 beam 步进 `setFree`，刷新时间戳。
2. **占据**：hit 且在有效距离、不在自身 footprint 内 → `setOccupied(100)`。
3. **Footprint 自滤**：径向半径或矩形 `length/width + margin`，避免机身回波成假障碍。

### 盲区如何被记住

前向约 90° 看到的障碍写入 occupied 并刷新时间戳；机器人转走后，盲区格不再被射线刷新，仍保持占用，直到超时衰减。若之后有可见自由射线穿过，则**立即清空**，不必等衰减。

`/scan_360`：从机体/传感器帧向栅格投 360 根射线，把记忆障碍投影成虚拟全向激光，供 NeuPAN 使用；遇到 unknown 格通常停止并报无效距离，避免假障碍环。

### 记忆时长：定时，非永久

| 单元 | 行为 |
| --- | --- |
| Occupied（盲区障碍） | `(now - cell_stamp) > cell_decay_sec` → 回 unknown |
| Free | **不衰减**（保留可通行射线） |
| 可见自由射线扫过的 occupied | **立即清空** |
| 部署默认 | **`cell_decay_sec = 8.0` s** |

结论：**不是永久盲区记忆，默认约 8 秒衰减记忆**；`cell_decay_sec ≤ 0` 会关闭衰减，文档警告可能堆幽灵障碍。

实现位置（上游仓）：`ros_ws/src/sliding_map/`、`ros2_ws/src/sliding_map/`；架构说明见 ONS2 `docs/architecture/`。

---

## Q2. 视线捷径（LOS）与 Laplacian 平滑

ONS2 `map_planner` 流水线：

`8-connected A*` → **`simplifyPath`（LOS 剪枝）** → 密化 → **`smoothPath`（Laplacian）** → 校验

### LOS shortcut：是一种 A\* 后的剪枝 / string-pull

- 不是搜索过程中的启发式剪枝，而是路径后处理：从当前点向后找**最远仍通视**的点，跳过中间折点（贪婪 funnel / string-pull）。
- 通视带安全间隙：例如 `line_of_sight_safe_distance`（约 0.25–0.35 m，按机型）与 `path_min_clear_m` 等，要求线段附近对障碍保持 clearance。

### Laplacian 平滑

- 端点固定，中间点做 Jacobi 式邻域平均，例如  
  \(c = 0.25\,p_{i-1} + 0.5\,p_i + 0.25\,p_{i+1}\)。
- 迭代次数约 20–40（`smoothing_iterations`），相对原路径偏差不超过 `smoothing_max_deviation_m`；清障/clearance 变差或超差则回滚。
- 作用：在 LOS 剪枝后的折线基础上进一步降折角、提高可跟踪性。

---

## Q3. SRU

### 一句话宏观总结

SRU 是带空间增强循环记忆的端到端深度导航策略：直接把深度图与目标映射为机体速度命令，绕过全局 A\* 与基于 `/scan_360` 的局部规划。

### 边界：ONS2 接线 vs SRU-Odin 模型

| 在 ONS2 | 在 [SRU-Odin](https://github.com/ManifoldTechLtd/SRU-Odin)（外部仓） |
| --- | --- |
| `--local-planner=sru` 启动与话题 remap | ONNX 策略 / VAE 编码器与推理节点 |
| goal → `/sru/goal_pose`；`/sru/cmd_vel` 经 enable 门控到 `/cmd_vel` | 张量 I/O、控制频率、`policy_scale` |
| 到达阈值 `sru_arrive_threshold_m`（默认约 0.7 m） | 训练与 `Deployment/` 模型文件 |

### 输入（话题 + 含义）

| 话题（部署侧常见名） | 含义内容 |
| --- | --- |
| 深度图（如 `/odin1/depth_img_competetion`） | 前视深度，表示前方几何与可通行通道；经 clip / resize（如约 `(40,64)`）后进入 VAE/策略 |
| 里程计（如 `/odin1/odometry_highfreq`） | 机体相对运动与位姿反馈，供策略状态与目标相对化 |
| `/sru/goal_pose` | 当前航点目标位姿（与 odom 同坐标系）；ONS2 任务层直接下发，**不经** `/initial_path` |

### 输出（话题 + 含义）

| 话题 | 含义内容 |
| --- | --- |
| `/sru/cmd_vel` | 机体坐标系速度命令（如 `[vx, vy, ωz]`），典型约 **5 Hz**；经 `policy_scale` 等缩放后输出 |
| `/sru/cmd_vel_enable` | 任务层门控：为真时 relay 才把策略速度转发到 `/cmd_vel`，保证与 `atomic_move` 互斥 |
| `/cmd_vel`（经 relay） | 最终底盘速度接口 |
| 到达判定（mission） | 相对目标平面距离小于约 **0.7 m** 视为航点到达 |

模型张量细节（属 SRU-Odin，非 ONS2 源码）：观测向量 + LSTM 隐状态 → 动作与新隐状态；障碍感知走**深度图**，不是激光栅格。

---

## Q4. NeuPAN

### 一句话宏观总结

NeuPAN 是罚函数型滚动时域 MPC：用解析或神经 DUNE 把障碍点编成碰撞约束特征，再在潜空间用 SCS 求解 NRMP，直接输出速度命令以跟踪全局路径并避障。

### 输入（话题 + 含义）

| 话题 | 含义内容 |
| --- | --- |
| `/scan_360`（或 `/scan`） | 虚拟/真实激光扫描，表示周围障碍点云；来自 sliding_map 时可覆盖盲区记忆 |
| `/initial_path` | 全局规划给出的参考折线路径，供 MPC 跟踪 |
| Action `/neupan/nav_path` | 任务层下发的本段跟踪目标路径（与 mission 状态机对接） |
| `/odin1/odometry` 与 TF | 当前位姿与坐标系变换（`map` / `odom` / `base_link`），用于状态与障碍坐标统一 |
| `/neupan/stop` | 任务层停止信号：NeuPAN 停发 `/cmd_vel`，把控制权交给对齐等模块 |

### 输出（话题 + 含义）

| 话题 | 含义内容 |
| --- | --- |
| `/cmd_vel` | 底盘线速度 / 角速度命令（差速等运动学下为规划器主输出） |
| `/neupan/replan` | 请求上层重新规划（如脱困 `stuck_escape` 结束后） |
| `/neupan/arrive` | 到达路径终点的通知 |
| `/neupan/control_trace`、`/neupan/escape_trace` | 调试轨迹：MPC / 自适应 / 脱困门控状态 |

### 实现要点

| 问题 | 说明 |
| --- | --- |
| 迭代 / 频率 | ROS 控制环约 **50 Hz**；MPC 时域 `receding`（常约 10）、`step_time`（约 0.1–0.2 s）；PAN 内层迭代约 2 次；Jetson 单次规划常见约 30–40 ms |
| 是否优化 | **是**：滚动时域罚函数 MPC / QP |
| 求解器 | 部署侧常用 **SCS**（cvxpy 建模；由 ECOS 迁来），可热启动 |
| 障碍处理 | **以软约束 / 惩罚为主**：距离松弛 `indep_dis∈[d_min,d_max]`、推开罚 `ro_obs`、净空奖励 `η`；**不是**经典硬安全走廊。外环另有硬停：`min_distance < collision_threshold` → 零速 |
| DUNE | 出厂常 `dune_use_analytic: true`，用闭式映射替代每拍 MLP；否则加载 `dune_checkpoint` |
| 每帧管线 | （可选）`adaptive_d_max` → `stuck_escape`（可覆盖 MPC）→ 否则 DUNE+NRMP → 加速度限幅 / 死区 → 发 `/cmd_vel` |

配置真源：`robots/<robot>/neupan/{config,planner}.yaml`；核心库在仓库内 `NeuPAN/`。

---

## 与本工程的价值

- 提供「感知定位 → 障碍记忆 → 全局 A\* → 局部 MPC/学习策略 → 任务交接 → 低层运控」的完整导航接口样板，适合对照四足 `cmd_vel` 接口与 [SEA-Nav](../论文/初步阅读/SEA-Nav.md) 一类导航策略。
- sliding_map 的定时盲区记忆、LOS+Laplacian 全局后处理、NeuPAN 软约束 MPC 与 SRU 端到端深度策略，可作为 02 路线的工程对照案例。

## 参考链接

- ONS2 仓库：<https://github.com/ManifoldTechLtd/Odin-Nav-Stack2>
- NeuPAN 上游思路：<https://github.com/hanruihua/NeuPAN>
- SRU-Odin：<https://github.com/ManifoldTechLtd/SRU-Odin>
- Odin1 驱动：<https://github.com/manifoldsdk/odin_ros_driver>
