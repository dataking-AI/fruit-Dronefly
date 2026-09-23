---
tags:
  - fruit-dronefly
  - 工程审查
类型: 工程改动
工程状态: 局部 reset 已修复
文档审查状态: 已完成本轮静态审查
代码实施状态: 局部 reset 修复已提交 f7f085d；CUDA 优化已提交 f473d7b
运行验证状态: 4 环境局部 reset、CUDA 采样和 PPO smoke 专项通过；Lee 时序未改变
核查日期: 2026-09-22
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

DroneAction.reset_idx 仅清理选中环境的动作、延迟缓冲和控制器状态，保留选中环境当前 yaw；force／torque 清零并按 env_ids 写入 composer，不在 reset 内重算全批悬停推力。下一策略步正常计算 Lee 控制输出。

2026-09-21 已补齐命令路径的索引隔离和历史推进修复，并通过专项回归。原缺陷分析保留在下文历史记录中；实际验证范围与限制见本页最后的修复与验收记录。

## 改动记录与验收

- **源码位置**：Lee.compute／compute_from_vel，DroneAction.process_actions／apply_actions／reset_idx，v0.__post_init__。
- **当前行为**：速度误差经 Lee 转换为推力力矩，物理子步施加缓冲。
- **计划操作**：保留该链路和基类依赖；删除未使用 rotor 调参路径，不重新设计低层控制。
- **改后行为**：换 Actor 后仍由传统控制器稳定姿态。
- **关联影响**：时序、动作延迟、质量惯量、contact 更新、局部 reset。
- **验收条件**：核对每环境步控制计算／每物理步施力调用次数；零速度指令能维持悬停趋势；无 NaN；局部 reset 不扰乱未重置环境状态。
- **审查状态**：局部 reset 与运行调用次数专项验证通过；长期训练及部署验证不在本次范围。

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
| [source/fruit_dronefly/fruit_dronefly/common/controller/lee_controller_position_and_yaw.py](<D:/zerozero_code/fruit-dronefly/source/fruit_dronefly/fruit_dronefly/common/controller/lee_controller_position_and_yaw.py>) | 已存在；2026-09-21 静态核查 |
| [source/fruit_dronefly/fruit_dronefly/common/controller/lee_controller_position_and_yaw_cfg.py](<D:/zerozero_code/fruit-dronefly/source/fruit_dronefly/fruit_dronefly/common/controller/lee_controller_position_and_yaw_cfg.py>) | 已存在；2026-09-21 静态核查 |
| [source/fruit_dronefly/fruit_dronefly/common/actions/action.py](<D:/zerozero_code/fruit-dronefly/source/fruit_dronefly/fruit_dronefly/common/actions/action.py>) | 已存在；2026-09-21 静态核查 |
| [source/fruit_dronefly/fruit_dronefly/tasks/fruit_dronefly/fruit_dronefly_env_cfg.py](<D:/zerozero_code/fruit-dronefly/source/fruit_dronefly/fruit_dronefly/tasks/fruit_dronefly/fruit_dronefly_env_cfg.py>) | 已存在；2026-09-21 静态核查 |

## 局部 reset 复核与参考做法

以下为修复前历史记录。2026-09-20 比对当时仓库与原 D:\zerozero_code\dronefollow：common/actions/action.py、mpc_action.py，以及两个任务目录的 mdp/commands/commands.py，四个文件逐一 SHA-256 一致。两份 commands.py 彼此也一致。这是原工程当前实现中已有的缺陷，被复制到 FruitDronefly；不是 MPC 算法或 Isaac Lab 框架的必然限制。

- 普通动作 reset（迁移前历史行为） 与 MPC reset（历史路径：source/DroneFollow/DroneFollow/common/actions/mpc_action.py，已迁移或删除） 都全量重建机身推力／力矩缓冲，只填回 env_ids，然后全量 apply_actions。MPC v0/v1 均配置此动作类。
- 普通命令 reset（迁移前历史行为） 与 MPC 命令 reset（历史路径：source/DroneFollow/DroneFollow/tasks/dronefollow_mpc/mdp/commands/commands.py，已迁移或删除） 都在局部 reset 时调用 _update_target_pose，后者全量追加历史；正常环境步又追加一次。固定帧数延迟因此会受其他环境 reset 影响。MPC v0/v1 同样启用 0.11 s 目标延迟。

参考 [[06 本地代码库与复用清单#OmniDrones 控制与局部 reset 参考]]，计划采用以下局部修复原则，不直接移植其旧版仿真 API：

1. 常驻缓冲初始化一次；reset 仅原地更新选中行，保留其余环境的 force、torque、processed_actions 和控制状态。避免局部 reset 触发带副作用的全 batch 控制计算。
2. 核查 permanent_wrench_composer 的 env_ids 语义；优先只写被重置环境，或在全量写入时保证未重置行保持不变。旋翼局部坐标与机身坐标另行核对。
3. 目标轨迹更新与历史推进分离：正常环境步只追加一次；reset 只重填选中环境的各历史槽，不推进其他环境时间轴，也不清除其噪声样本。
4. 用两个以上环境固定状态／动作，比较其他环境不 reset 与频繁 reset 两组：目标历史、施力缓冲及下一物理步一致；另外覆盖全量 reset、首次 reset、连续 reset 和动作延迟。

历史状态（2026-09-20）：缺陷与原工程一致，尚未修复。2026-09-21 补修结果见下文；MPC 已删除，不作运行验收。

## 2026-09-21 局部 reset 修复与专项验收

本次用户授权修复 P01／P02 并更新状态。参考 OmniDrones `9ce7c2028b71be64d7e748c31f685cd3b54afe27` 的 `MultirotorBase._reset_idx`：按 env_ids 原地清理；未复制其旋翼 throttle 或旧仿真 API，未修改、安装或运行参考仓库。

- **原缺陷**：reset 调用 `_update_target_pose(env_ids)` 后仍全批追加历史、覆盖其他环境的噪声；课程切换还会重建整组轨迹并丢失其他环境的航点／方向。此前“P01 代码已修”的结论不完整，本条记录实际补修。
- **当前实现**：目标轨迹仅计算并写入选中行，approach 事件也按索引更新；每次 `_update_command` 只追加一次历史，reset 只覆写选中行。轨迹重新分组时保留继续运行环境的状态，未变化的组复用原对象。空 reset 不做操作，有限范围 slice 保持索引范围。动作 reset 不再刷新整批状态快照；Lee 状态、延迟动作、force／torque 及 permanent_wrench_composer 仅清理选中环境。
- **专项验证**：`env_isaaclab`／Isaac Sim 5.1，4 环境、19 维 Actor／Critic、3 维动作，`action_lag=2`，开启 approach，使用本地材质。七种轨迹的选中计算与完整计算一致；实际环境验证重复、空、slice、全量 reset，以及手动切换地形 level 后其他环境的轨迹、噪声、历史、动作、控制器和 composer 完全不变。
- **对照**：25 步无 reset／频繁 reset 的命令对照，在固定未来噪声采样后未重置行及延迟历史逐值一致。实际下一物理子步对照还原 root／joint 状态，未重置环境误差满足 `atol=2e-4, rtol=1e-4`。共享随机数发生器会因 reset 消耗随机数，本测试将其与持久状态污染分开检查，不承诺跨分支的未来随机样本天然相同。
- **时序与烟测**：连续 100 步，无 NaN／形状变化；每策略步一次目标历史追加、一次 Lee compute、四次全 batch 施力，局部 reset 可额外写选中行。持续环境的 last_action 等于上一策略命令。静态编译和 CRLF 感知的 Git 空白检查通过。
- **复现**：从源码仓库、env_isaaclab 执行 `python scripts/tests/check_partial_reset.py`。脚本：[check_partial_reset.py](<D:/zerozero_code/fruit-dronefly/scripts/tests/check_partial_reset.py>)。本机日志：[partial_reset_2026-09-21.log](<D:/zerozero_code/fruit-dronefly/logs/validation/partial_reset_2026-09-21.log>)，Git 忽略。
- **限制**：验证手动 level 切换的状态保持，未验证完整课程晋级策略；下一物理步对照不等于长时轨迹逐位确定性。首次局部 reset 专项当时未跑 PPO；后续 CUDA 优化已补做 3 次 PPO smoke。默认 4096 环境、远程材质、play／导出仍未验证。局部 reset 修复现已包含在 f7f085d，CUDA 优化已提交 f473d7b。图 Actor、FlyDrones 和新 checkpoint 格式仍未实施。

## 2026-09-22 CUDA 采样优化与控制时序边界

本轮优化位于 reset、轨迹朝向和命令分组刷新路径，没有改变控制器输入、调用频率或施力顺序。专项仍确认每策略步一次 Lee compute、每物理子步施力，即策略／Lee 25 Hz、物理 100 Hz。不能把局部采样 A/B 写成飞行品质或长期吞吐结论。

本轮在优化后复跑局部 reset 物理回归和 PPO smoke，详细结果及限制见 [[05 实施顺序与验收#CUDA 采样优化专项（P12）]]。


## 2026-09-23 P03：理想执行模型验证

用户将电机/旋翼可达性暂缓，当前按理想机体推力/力矩接口验收。已修正 Lee 陀螺补偿为力矩空间的 ω×Jω，并在零合力和推力方向与 heading 共线时回退到合法旋转矩阵；保持原控制增益、速度/加速度/总推力约束和 25 Hz/100 Hz 时序。CPU/CUDA 各 512 状态解析检查、12 场景修改前后闭环对照（24 环境、12 s）及原 4 环境局部 reset 回归通过；两个进程退出码 0。末段平均速度向量误差最大约 0.0001852 m/s，进入并保持 0.1 m/s 误差范围最慢约 1.44 s。修改前也通过这些速度场景，不能把此次公式修正宣称为显著改善常规速度跟踪。组合速度反向时实际单轴峰值约 5.49 m/s；速度指令限幅不是实际速度硬限幅。功能改动已提交 `acfdfcf`（2026-09-23 复核）；历史受测状态、详细覆写、限制与日志见[验证证据](<D:/zerozero_code/fruit-dronefly/logs/validation/p03_control_20260923/summary.md>)。

复现：`python scripts/tests/check_lee_control.py`，随后 `python scripts/tests/check_partial_reset.py`。电机能力、PPO 重训和任意状态全局稳定性不在本次通过范围。
