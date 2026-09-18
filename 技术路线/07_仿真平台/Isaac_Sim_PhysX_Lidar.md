# Isaac Sim PhysX Lidar 实战

原文链接：[Isaac Sim 激光雷达实战：从官方示例到速腾 128 线对齐](https://mp.weixin.qq.com/s/biKBVUyNkxG1Zt8GsuDbTA)

## 一句话定位

这篇文章围绕 Isaac Sim 的轻量级 PhysX SDK Lidar，整理了从官方 GUI 示例、手动挂载、速腾 RS-Ruby Plus 128 线参数等效，到 Python 读取深度/点云/语义数据的完整使用路径。

## 传感器选择

| 方案 | 特点 | 适用场景 |
| --- | --- | --- |
| PhysX SDK Lidar | 基于 PhysX 射线投射，只检测 Collider，输出 ground truth 距离，不模拟材质、反射、噪声和衰减 | 原型验证、导航策略输入、轻量传感器仿真 |
| RTX Lidar | 基于渲染管线，可模拟更真实的材质反射和噪声，但性能开销更高 | 高保真传感器仿真、非均匀线束和更接近真机的感知测试 |

PhysX Lidar 的核心限制是“只认碰撞体”。没有 Collider 的物体不会被测到；透明物体只要有碰撞体，也会被测到。

## GUI 使用路径

| 步骤 | 操作 | 目的 |
| --- | --- | --- |
| 运行官方示例 | `Windows > Examples > Robotics Examples > Sensors > Physx Lidar Sensor`，依次加载雷达、场景和源码后按 `PLAY` | 零配置查看扫描线效果 |
| 手动创建雷达 | `Create > Physics > Physics Scene`，再 `Create > Sensors > PhysX Lidar > Rotating` | 理解雷达 prim、物理场景和参数位置 |
| 添加碰撞体 | 给目标物体添加 `Physics > Collider` | 验证 PhysX Lidar 只检测碰撞几何 |
| 挂载到几何体或机器人 | 将 LIDAR prim 拖到父物体或机器人 link 下，并设置局部位姿 | 模拟真实安装位和坐标系 |

Isaac Sim 6.0.1 中，示例按钮名可能显示为 `Load LIDAR`、`Load LIDAR Scene` 和 `Open in IDE`。

## 速腾 RS-Ruby Plus 128 线等效参数

PhysX Lidar 的线束数按 `FOV / resolution + 1` 计算，两端边界都会生成一条线。要等效约 128 线，可使用 `vertical_fov=40`、`vertical_resolution=0.3125`，得到约 129 条垂直线束。

| 真机规格或目标 | PhysX Lidar 参数 | 说明 |
| --- | --- | --- |
| 128 线 | `vertical_fov=40`，`vertical_resolution=0.3125` | 40 / 0.3125 + 1 = 129，近似 128 线 |
| 垂直视场 `-25°~+15°` | `vertical_fov=40`，雷达绕 Y 轴约 `+5°` | PhysX 默认垂直通道近似对称分布，需俯仰补偿 |
| 水平 360° 环扫 | `horizontal_fov=360` | 与机械式环扫一致 |
| 水平分辨率 0.2° | `horizontal_resolution=0.2` | 每圈约 1800 点 |
| 帧率 10 Hz | `rotation_rate=10` | 转速可理解为扫描帧率；`0` 表示一次性全向发射 |
| 量程与盲区 | `min_range=0.4`，`max_range=250` | 对齐真机近距离盲区和远距量程 |
| 多线显示 | `high_lod=True` | 不开启时可能退化成单线显示 |

需要注意，RS-Ruby Plus 的真实垂直通道并非完全均匀排布，PhysX Lidar 只能做均匀线束近似；要模拟非均匀通道和更真实反射特性，应转向 RTX Lidar 配置路线。

## 推荐配置

| 用途 | 参数 |
| --- | --- |
| 演示与调试 | `vertical_resolution=0.3125`，`horizontal_resolution=1.0`，`rotation_rate=2`，`min_range=0.4`，`max_range=100`，`high_lod=True` |
| 接近真机点率 | `vertical_resolution=0.3125`，`horizontal_resolution=0.2`，`rotation_rate=10`，`min_range=0.4`，`max_range=250`，`high_lod=True` |

真机点率可粗略按 `128 线 * 1800 点/圈 * 10 Hz` 估算，约 230 万点/秒。

## Python 取数要点

脚本侧通过 `_range_sensor.acquire_lidar_sensor_interface()` 获取接口，常用读取函数包括：

| 函数 | 输出 |
| --- | --- |
| `get_linear_depth_data(path)` | 每个点的线性距离 |
| `get_zenith_data(path)` | 俯仰角 |
| `get_azimuth_data(path)` | 方位角 |
| `get_point_cloud_data(path)` | XYZ 点云 |
| `get_prim_data(path)` | 命中的 prim 或语义 ID |

读取前通常需要启动 timeline 并等待至少一帧，让雷达缓冲区生成数据；如果启用语义分割，创建雷达时需要设置 `enable_semantics=True`，并给目标 prim 添加 `SemanticsAPI` 标签。

## 避坑清单

| 现象 | 处理 |
| --- | --- |
| 只显示一条线 | 检查 `high_lod=True`，否则密集波束可能退化为单线显示 |
| 运行中改 FOV 或分辨率无效 | 波束模式在 `PLAY` 启动时生成，先 `STOP`，改参数后再 `PLAY` |
| 雷达看不到物体 | 检查目标物体是否添加 `Collider` |
| 扫描线在浅色地面不明显 | 使用深色场景或调试材质，必要时用 `rotation_rate=0` 显示全向发射效果 |
| 示例重载导致参数丢失 | 调好参数后不要再次点击示例加载按钮 |
| 挂载旋转不生效 | 确认 xformOp 已进入 `xformOpOrder`；GUI 视口旋转通常会自动登记 |

## 与足式机器人路线的关系

- 用于 [02 基础感知与导航](../02_基础感知与导航/README.md)：给导航策略提供 LiDAR、点云和障碍物输入。
- 用于 [03 Locomotion](../03_Locomotion/README.md)：在复杂地形或障碍环境中验证感知闭环 Locomotion。
- 用于 [04 Sim2Real与实机部署](../04_Sim2Real与实机部署/README.md)：对齐真机传感器视场、频率、安装位和数据接口，降低仿真到实机的传感器差异。
