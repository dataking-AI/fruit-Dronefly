---
tags:
  - fruit-dronefly
  - 工程审查
类型: 工程改动
工程状态: 未实施
文档审查状态: 已完成本轮静态审查
代码实施状态: 未实施
运行验证状态: 未执行
核查日期: 2026-09-20
文档修订: 独立接手版（最小骨架与FlyDrones扩展）
源码根目录: 'D:\zerozero_code\fruit-dronefly'
---

# Lee 控制器与执行时序

[[07 工程上下文与接手约定|工程上下文与接手约定]] · [[README|阅读入口]]

[[03 观测动作与控制接口/00 接口改动总览|返回接口总览]] · [[03 观测动作与控制接口/03 坐标转换与偏航来源|上游坐标]]

## 保留配置与调用链

| 项目 | 当前 v0／首版计划 |
|---|---|
| sim.dt | 0.01 s，物理 100 Hz |
| decimation | 4，环境／策略周期 0.04 s，即 25 Hz |
| controller dt | env.cfg.sim.dt × decimation，即 0.04 s |
| Lee 输入 | 世界速度 m/s＋期望 yaw rad；yaw 不是 yaw_rate |
| k_v | [3.8,3.8,6.0] |
| 惯量来源 | sim_composed，保留质量与组合惯量获取 |
| render_interval | 4；这不表示已配置机载相机 |
| episode_length_s | 30 s |
| action_lag | 0，代码仍保留延迟机制 |

~~~mermaid
sequenceDiagram
    participant P as 策略
    participant A as DroneAction
    participant L as Lee
    participant S as 仿真
    P->>A: 每环境步 raw action
    A->>A: 缩放、坐标转换、偏航
    A->>L: process_actions 中 compute
    L-->>A: 推力与力矩缓冲
    loop 4 个物理子步
        A->>S: apply_actions 施加缓冲
        S->>S: 物理积分
    end
~~~

这是当前项目代码对应的预期调用关系，实施时需与安装版 ManagerBasedRLEnv 调用栈核对。当前 Lee 在 process_actions 计算，apply_actions 主要重复施力；不能把 100 Hz 物理积分写成 Lee 100 Hz 闭环更新。

## reset 保留与核查

DroneAction.reset_idx 清理选定环境的原始动作、命令历史和延迟，保留当前 yaw，调用 controller.reset_idx，并重建悬停所需推力缓冲。命令历史另由轨迹模块处理。

静态复核已确认 reset 全量重建 force_body／torque_body，只回填选中环境后全量施力，会清空未重置环境的缓冲。下一次 process_actions 可能覆盖，因此实际轨迹影响仍须物理步回归，不能直接断言必然掉高。普通与 MPC 分支均有同样缺陷；详见下方“局部 reset 复核与参考做法”。

## 改动记录与验收

- **源码位置**：Lee.compute／compute_from_vel，DroneAction.process_actions／apply_actions／reset_idx，v0.__post_init__。
- **当前行为**：速度误差经 Lee 转换为推力力矩，物理子步施加缓冲。
- **计划操作**：保留该链路和基类依赖；删除未使用 rotor 调参路径，不重新设计低层控制。
- **改后行为**：换 Actor 后仍由传统控制器稳定姿态。
- **关联影响**：时序、动作延迟、质量惯量、contact 更新、局部 reset。
- **验收条件**：核对每环境步控制计算／每物理步施力调用次数；零速度指令能维持悬停趋势；无 NaN；局部 reset 不扰乱未重置环境状态。
- **审查状态**：工程未实施；运行调用次数与 reset 回归待验证。

- [ ] 不将本次骨架通过等同真实无人机控制验证。
- [ ] 记录 controller 输出的单位及所在坐标系。
- [ ] 部署时重新核对控制周期，见 [[04 后续升级/05 部署与控制接口适配]]。

## 与 FlyDrones 运行主循环的边界

FlyDrones runtime 默认按约 20 Hz 调用单个 Pilot，Brain 内部又按 LIF 积分时间步演进；其简化 SimDrone 用一阶速度响应模拟飞行。两者均不迁入本项目首版。

继续使用本页 25 Hz 环境步、100 Hz 物理步和原 Lee 调用链。未来连续值回路的神经更新子步要与环境时间显式对齐，不能把上游 0.5 ms 脉冲步长直接当成项目控制周期。数据资产复用不引入这些时间参数。

## 已存在源码入口

以下是核查时实际存在的文件；正文中的“拟新增”文件不在此表冒充现状。

| 源码位置（相对代码仓库） | 依据 |
|---|---|
| [source/DroneFollow/DroneFollow/common/controller/lee_controller_position_and_yaw.py](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/common/controller/lee_controller_position_and_yaw.py>) | 已存在；2026-09-20 静态核查 |
| [source/DroneFollow/DroneFollow/common/controller/lee_controller_position_and_yaw_cfg.py](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/common/controller/lee_controller_position_and_yaw_cfg.py>) | 已存在；2026-09-20 静态核查 |
| [source/DroneFollow/DroneFollow/common/actions/action.py](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/common/actions/action.py>) | 已存在；2026-09-20 静态核查 |
| [source/DroneFollow/DroneFollow/tasks/dronefollow/dronefollow_env_cfg_v0.py](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/tasks/dronefollow/dronefollow_env_cfg_v0.py>) | 已存在；2026-09-20 静态核查 |

## 局部 reset 复核与参考做法

2026-09-20 比对当前仓库与原 D:\zerozero_code\dronefollow：common/actions/action.py、mpc_action.py，以及两个任务目录的 mdp/commands/commands.py，四个文件逐一 SHA-256 一致。两份 commands.py 彼此也一致。这是原工程当前实现中已有的缺陷，被复制到 FruitDronefly；不是 MPC 算法或 Isaac Lab 框架的必然限制。

- [普通动作 reset](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/common/actions/action.py>) 与 [MPC reset](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/common/actions/mpc_action.py>) 都全量重建机身推力／力矩缓冲，只填回 env_ids，然后全量 apply_actions。MPC v0/v1 均配置此动作类。
- [普通命令 reset](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/tasks/dronefollow/mdp/commands/commands.py>) 与 [MPC 命令 reset](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/tasks/dronefollow_mpc/mdp/commands/commands.py>) 都在局部 reset 时调用 _update_target_pose，后者全量追加历史；正常环境步又追加一次。固定帧数延迟因此会受其他环境 reset 影响。MPC v0/v1 同样启用 0.11 s 目标延迟。

参考 [[06 本地代码库与复用清单#OmniDrones 控制与局部 reset 参考]]，计划采用以下局部修复原则，不直接移植其旧版仿真 API：

1. 常驻缓冲初始化一次；reset 仅原地更新选中行，保留其余环境的 force、torque、processed_actions 和控制状态。避免局部 reset 触发带副作用的全 batch 控制计算。
2. 核查 permanent_wrench_composer 的 env_ids 语义；优先只写被重置环境，或在全量写入时保证未重置行保持不变。旋翼局部坐标与机身坐标另行核对。
3. 目标轨迹更新与历史推进分离：正常环境步只追加一次；reset 只重填选中环境的各历史槽，不推进其他环境时间轴，也不清除其噪声样本。
4. 用两个以上环境固定状态／动作，比较其他环境不 reset 与频繁 reset 两组：目标历史、施力缓冲及下一物理步一致；另外覆盖全量 reset、首次 reset、连续 reset 和动作延迟。

状态：缺陷源码路径与原工程一致性已确认；OmniDrones 仅静态参考；本轮未修复功能代码、未运行 MPC 或物理轨迹回归。
