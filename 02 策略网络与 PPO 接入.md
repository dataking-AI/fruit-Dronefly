---
tags:
  - fruit-dronefly
  - 工程审查
类型: 工程改动
工程状态: 未实施
文档审查状态: 已完成本轮静态审查
代码实施状态: 基线 acfdfcf 加未提交 F01 模块；首版测试图及相机闭环未实施
运行验证状态: 原有专项及 F01 独立探针通过；完整飞行策略与相机闭环验收仍待完成
核查日期: 2026-09-20
文档修订: 独立接手版（最小骨架与FlyDrones扩展）
源码根目录: 'D:\zerozero_code\fruit-dronefly'
---

# 策略网络与 PPO 接入

## 2026-09-23 F01 当前进展

F01 固定视频、冻结状态、读出 PPO 与导出已通过；相机避障闭环未接入。源码基线 `acfdfcf` 加未提交 F01 工作树，不能仅凭 HEAD 复现；12 个受测文件 SHA-256 见 [[05 实施顺序与验收#2026-09-23 F01：冻结视觉独立探针与读出验证]]。环境为从 `env_isaaclab` 克隆的 `fruit_dronefly`。本页既有首版与历史验收记录保留，独立读出验证不替代测试图、完整飞行策略或相机闭环验收。


[[07 工程上下文与接手约定|工程上下文与接手约定]] · [[README|阅读入口]]

[[00 改动总览|返回总览]] · [[03 观测动作与控制接口/00 接口改动总览|接口审查]]

## 待新增模块

以下路径均为拟新增，当前不存在：source/fruit_dronefly/fruit_dronefly/policies/ 下的测试图定义、稀疏 Actor 模型与配置。已确认 RSL-RL 5.0.1 使用独立 actor/critic 模型及原生模块路径解析；不再计划新增旧版一体化 ActorCritic 或兜底 factory。已确定的包名迁移决策另见清理页，不能把拟新增类当成已存在 API。

| 改动项 | 当前 | 计划改后 |
|---|---|---|
| Actor | 普通 MLP [32,32]，ELU | 极小分层有向无环测试图；连续值；无跨步状态 |
| 图拓扑 | 无 | 固定节点／边清单与输入输出 ID；版本化保存 |
| 权重 | 全连接权重 | 仅存在边有效，训练边权和偏置；掩码注册为非训练 buffer |
| 输出 | 3 维策略动作 | 3 维均值和合法标准差；训练采样、回放用确定性均值 |
| Critic | MLP [32,32] | 保留独立 MLP，不与测试图共享权重 |
| 算法 | RSL-RL PPO | 保留 PPO 与首版超参数；只替换策略结构 |

每层按拓扑顺序计算 masked linear＋非线性，中间层连续激活，最终均值读出为线性。不能用一次方阵乘法暗示所有多跳路径均完成传播。第一版的具体节点数与连边表属于测试夹具，应固定并写入模型配置；不是待提取的真实果蝇数据。

## 调用接口

| 接口边界 | 首版契约 |
|---|---|
| actor 输入 | B×19，固定顺序，见观测页面 |
| critic 输入 | B×19 独立观测组；实际噪声边界见观测页面 |
| 策略输出 | 原始动作 B×3；速度缩放只发生在 action 层 |
| value | 对外语义 B×1；适配安装版本所需形状 |
| rollout | 无记忆流程；不修改 storage 或引入 BPTT |
| reset | Actor 无跨步状态；环境／命令／控制器状态仍正常 reset |

保留 rollout=16、ELU、init_noise_std=0.5、learning_rate=4e-4、gamma=0.98、lambda=0.95 等原 PPO 设置作为首版起点。两种 Actor 在同一任务、奖励和训练预算下运行；本阶段验收接口与训练稳定性，不宣称测试图有生物学优势。

## 训练入口与版本核查

已通过 env_isaaclab 的实际元数据、导入路径与原 v0 运行核查，详细版本和证据见 [[08 运行环境兼容性与版本方案]]。当前采用 rsl-rl-lib 5.0.1；源码 train.py 的 3.0.1 是最低门槛，不是本项目目标版本。

- 使用 actor/critic 两个原生模型配置；图 Actor 实现或继承 MLPModel 协议，Critic 保持独立 MLPModel [32,32]。
- obs_groups 显式设为 actor→policy、critic→critic；分布配置 GaussianDistribution(init_std=0.5,std_type=scalar)，保持原 PPO 数值。
- class_name 使用模块:类完整路径，resolve_callable 已核查支持；不修改 PPO，不需要额外全局注册或 monkey patch factory。
- train/play 使用同一模型配置和可导入路径。原 MLP 配置可经 handle_deprecated_rsl_rl_cfg 转换，但自定义新图直接使用原生配置，不能依赖旧 policy 字段自动识别。
- Actor 必须满足随机／确定性 forward、分布与 log-prob、entropy/KL、无状态 reset/hidden state、normalization 以及 as_jit/as_onnx 接口。优先复用 MLPModel 通用方法，具体 mask 网络和导出仍须测试。
- MLP 与图两种配置使用现有 --agent 配置入口区分；具体新配置 entry point 随实现记录，不新增物理环境 ID。

## 保存、回放和导出

checkpoint 必须足以恢复策略类型、固定图／掩码、节点映射、参数及训练优化器状态；环境配置快照记录观测顺序、动作语义与图版本。原 MLP／MPC checkpoint 不自动兼容。

play 当前自动导出模型且文件名含 h151，拟改为新实验命名。新 Actor 需兼容实际版本导出路径；导出物必须含固定掩码，与 PyTorch 确定性输出数值一致。不预先声称任何稀疏算子均可导出，首版可用小规模稠密张量加掩码实现。

## 改动记录与验收

- **源码位置**：原 PPO 配置、train／play；policies 模块为拟新增。
- **当前行为**：原 policy 配置经兼容转换实例化两个 MLPModel，原 v0 PPO 短运行通过。
- **计划操作**：新增测试图及 Actor，适配注册、配置、保存、回放和导出。
- **改后行为**：PPO 正常更新存在边，非连接位置不参与有效计算。
- **关联影响**：观测固定索引、动作分布、checkpoint 识别、导出和日志命名。
- **验收条件**：输出形状正确；合适非退化输入下存在边有有限梯度；优化后有效非连接权重仍为零；保存恢复和导出结果一致。
- **审查状态**：图策略未实施；5.0.1 API 与原 v0 基线已核查，图策略合约仍待实现验收。

- [ ] 不将“某批次某条边梯度为零”直接判定为错误；验证图中有效路径与多组输入。
- [ ] 测试图与 MLP 的 critic、观测、动作和环境设置一致。
- [ ] 训练至少完成少量 PPO 更新并可回放新 checkpoint。

后续：[[04 后续升级/03 真实连接组替换测试图]] · [[04 后续升级/04 循环状态与 PPO 序列训练]]。

## 固定拓扑参数化与后续数据边界

首版实施基线 的 h=φ((M⊙W)x+b) 在首版解释为逐层计算：

$$
h^{(\ell+1)}=\phi_\ell\!\left[(M_\ell\odot W_\ell)h^{(\ell)}+b_\ell\right].
$$

M 为固定连边；W 为 PPO 更新的参数；最后一层线性读出三维高斯均值。不存在的边不参与有效计算，不用有循环图的一次矩阵乘法冒充多跳传播。

后续从 FlyDrones 接入的是图资产及节点分组，不是 LIFNetwork、Brain.tick 或 MotorDecoder。训练端把静态连边构造成独立可训练参数，原始突触数／递质符号作为来源数据保存，初始化后不覆盖来源。连续值回路有循环时依赖 [[04 后续升级/04 循环状态与 PPO 序列训练]]。

| 保留的外部接口 | 必须明确允许变化的内部接口 |
|---|---|
| PPO 算法、三维 raw action、DroneAction＋Lee | 节点数、边数、输入映射、状态与计算子步 |
| checkpoint 的 runner 保存／恢复机制 | 图配置、模型类型及参数形状；旧模型不能自动加载到新图 |
| 训练／play 入口的用途 | 5.0.1 原生模型路径、配置与导出适配 |

数据来源与三维输出读出分别见 [[04 后续升级/06 FlyDrones 连接组数据适配]] 和 [[04 后续升级/07 感觉运动节点映射与可训练读出]]。FlyDrones 的 calibrate 仅拟合 throttle／yaw 回归读出，不替代 PPO，也不作为首版训练前置。

## 已存在源码入口

以下是核查时实际存在的文件；正文中的“拟新增”文件不在此表冒充现状。

| 源码位置（相对代码仓库） | 依据 |
|---|---|
| [source/fruit_dronefly/fruit_dronefly/tasks/fruit_dronefly/agents/rsl_rl_ppo_cfg.py](<D:/zerozero_code/fruit-dronefly/source/fruit_dronefly/fruit_dronefly/tasks/fruit_dronefly/agents/rsl_rl_ppo_cfg.py>) | 已迁移；2026-09-22 路径复核，未新增运行证据 |
| [scripts/rsl_rl/train.py](<D:/zerozero_code/fruit-dronefly/scripts/rsl_rl/train.py>) | 已存在；2026-09-20 静态核查 |
| [scripts/rsl_rl/play.py](<D:/zerozero_code/fruit-dronefly/scripts/rsl_rl/play.py>) | 已存在；2026-09-20 静态核查 |

## F01 独立 Actor 与首版图策略的边界

新增 FlyVisActor 采用 RSL-RL 5.0.1 原生 MLPModel 接口，训练、保存及完整读出 JIT/ONNX 已通过。它是冻结视觉活动的可训练读出，不是本页拟实施的无记忆稀疏测试图；默认任务配置仍用原 MLPModel。具体输入缓存、参数与导出边界见 [[04 后续升级/07 感觉运动节点映射与可训练读出]]。
