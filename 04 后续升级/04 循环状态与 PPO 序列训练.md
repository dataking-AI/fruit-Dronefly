---
tags:
  - fruit-dronefly
  - 工程审查
类型: 后续升级
工程状态: 后续未实施
文档审查状态: 已完成本轮静态审查
代码实施状态: 后续未实施
运行验证状态: 未执行
核查日期: 2026-09-20
文档修订: 独立接手版（最小骨架与FlyDrones扩展）
源码根目录: 'D:\zerozero_code\fruit-dronefly'
---

# 循环状态与 PPO 序列训练

[[07 工程上下文与接手约定|工程上下文与接手约定]] · [[README|阅读入口]]

[[04 后续升级/00 升级依赖与实施边界|返回升级总览]] · [[02 策略网络与 PPO 接入|对应当前无记忆策略]]

## 状态接口变化

| 首版 | 后续动态回路 |
|---|---|
| obs → action | obs、h_t → action、h_next |
| 每次独立前向 | 每个并行环境独立保存状态 |
| 无序列缓存要求 | rollout 记录序列起点状态及 episode 边界 |
| Actor 无状态 reset | 仅按 dones/env_ids 清理对应状态 |
| 导出仅观测输入 | 导出需显式状态输入输出和初始化规则 |

FlyVis 若保留原有时间动力学，也会产生需要管理的视觉状态；“策略核心无记忆”不代表完整 actor 可以按无状态处理。

## 实施前核查而非预设重写

先检查实际 RSL-RL 版本已有 recurrent policy、storage、mask 和序列 minibatch 支持。若足够，复用其接口；只有接口缺口才补项目适配，不预先要求重写 rollout storage。play 已有版本相关 reset 分支，但自定义状态能否通过该分支正确清理仍需验证。

## 改动记录与验收

- **源码位置**：train／play／PPO 配置；后续修改自定义 Actor 的状态契约。
- **当前行为**：拟首版为无记忆测试图，尚未实施。
- **计划操作**：增加连续值状态更新、状态初始化、序列批处理、截断反传和 reset。
- **改后行为**：循环回路与气味／视觉历史可以参与策略决策。
- **关联影响**：采样、训练状态一致性、计算子步与控制周期、保存恢复、导出和部署。
- **验收条件**：分段与连续推理一致；局部 reset 不清其他环境；训练不跨 episode 串接梯度；checkpoint 恢复与状态规则可复现。
- **审查状态**：后续未实施；隐藏状态规模、时间常数及展开长度待模型选定后审查。

- [ ] 无跨 episode 的状态泄漏。
- [ ] 正确区分 episode 结束、time-limit bootstrap 和计算图截断。
- [ ] 不能把循环收益全部归给果蝇特有连边。
- [ ] 与 [[04 后续升级/05 部署与控制接口适配|部署]] 共享状态、频率及 reset 契约。

## FlyDrones 动态状态不直接迁移

本地 Brain.tick 的接口为“输入群→泊松频率、模拟毫秒数→群平均放电率”，其 NumPy LIF 状态不是 PyTorch 自动微分状态。仅转换数组为 Tensor 或给输出加 requires_grad 不能恢复内部梯度。

后续项目重新定义连续值 h_t 更新和时间尺度，从 [[04 后续升级/06 FlyDrones 连接组数据适配|图资产]] 读取有向边；复用拓扑，不继承脉冲阈值／不应期／0.5 ms 步长。FlyDrones 的 Brain.copy 能复制独立神经状态，但不等同 Isaac Lab 批量 B 维网络或 PPO 序列缓存实现。

## 已存在源码入口

以下是核查时实际存在的文件；正文中的“拟新增”文件不在此表冒充现状。

| 源码位置（相对代码仓库） | 依据 |
|---|---|
| [scripts/rsl_rl/train.py](<D:/zerozero_code/fruit-dronefly/scripts/rsl_rl/train.py>) | 已存在；2026-09-20 静态核查 |
| [scripts/rsl_rl/play.py](<D:/zerozero_code/fruit-dronefly/scripts/rsl_rl/play.py>) | 已存在；2026-09-20 静态核查 |
| [source/DroneFollow/DroneFollow/tasks/dronefollow/agents/rsl_rl_ppo_cfg.py](<D:/zerozero_code/fruit-dronefly/source/DroneFollow/DroneFollow/tasks/dronefollow/agents/rsl_rl_ppo_cfg.py>) | 已存在；2026-09-20 静态核查 |
