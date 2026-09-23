---
tags:
  - fruit-dronefly
  - 工程审查
  - FlyDrones
类型: 后续升级
工程状态: 后续未实施
文档审查状态: 已完成本轮静态审查
代码实施状态: 后续未实施
运行验证状态: 未执行
核查日期: 2026-09-22
文档修订: FlyVis 优先视觉 MVP，MaleCNS 第二路线
源码根目录: 'D:\zerozero_code\fruit-dronefly'
FlyDrones版本: '3e269346b3882c291d2a977bc2c2c6a9c9213c21'
---

# FlyDrones 连接组数据适配

[[07 工程上下文与接手约定|工程上下文与接手约定]] · [[README|阅读入口]]

[[04 后续升级/00 升级依赖与实施边界|返回升级总览]] · [[04 后续升级/03 真实连接组替换测试图|接入策略核心]] · [[06 本地代码库与复用清单|本地版本与来源]]

## 路线位置与回退

本页属于 FlyVis 视觉 MVP 后的 M1 第二路线；MaleCNS 下载、子图构建与跨模型 adapter 不再是首轮相机实验的前置条件。保留离线数据工具和候选分组复用，不实例化 LIF Brain，不复用固定 decoder。连续值核心训练超预算时可冻结已验证核心只训 readout；离线活动仅用于接口检查。

## 复用与新增边界

| 模块 | 上游位置／函数 | 项目计划 |
|---|---|---|
| 图容器与序列化 | brain/connectome.py：Connectome、save、load | 复用稀疏图与注释读取；输出项目图资产 |
| 神经元选择 | GroupSpec、select、resolve_groups | 复用类型正则＋侧别选择；补必需群检查和 roles |
| MaleCNS 加载 | build_malecns | 复用原始表读取和筛选；记录筛选、符号与数据版本 |
| 子图 | sensorimotor_core、subgraph | 复用算法；补路径、循环、预算和索引校验 |
| 配置 | defaults.yaml 的 inputs／outputs | 只取候选分组，不继承 LIF 参数或 decoder 行为 |
| 项目适配（拟新增） | 项目侧 tools／数据模块 | 独立离线流程，生成训练端可加载资产 |

## 离线流程

1. 固定本地 FlyDrones commit 和实际 MaleCNS 文件版本，确认数据并非已随仓库附带。
2. 调用 build_malecns 或读取已有 Connectome NPZ；检查节点、边和字段数量。
3. 从审查过的配置创建 GroupSpec，调用 resolve_groups；显式设置 meta["roles"] 的 input／output。
4. 必需感觉群或输出群为空时停止构建并报告名称／正则／侧别；可选群缺失可报告后继续，但不得静默把必需输入输出变成零。
5. 以 sensorimotor_core 提取候选图，必要时扩大 hops 或重新定义边界；subgraph 后核对节点重映射。
6. 导出版本化图资产及检查报告，供训练端独立加载。

> [!warning] 不照搬上游构建命令
> cli.cmd_build 通过 Brain(c,cfg) 设置角色，Brain 随即创建 LIFNetwork。仅 resolve_groups 不会自动填 meta.roles；项目适配直接填角色，不实例化 Brain，也不调用 calibrate／inspect／fly。数据工具依赖与训练运行依赖分开。

## 图资产最小契约（拟新增）

GraphAsset 是本项目拟议概念，不是 FlyDrones 已有类名或既定文件格式。

| 内容 | 语义 |
|---|---|
| node_ids／source_body_ids | 紧凑训练索引与原始 body_id 对应，不能混用 |
| cell_type／side／superclass | 分组和审查所需原始注释 |
| edge_src／edge_dst | 有向边源／目标；上游 W 采用 post,pre 排列 |
| synapse_count／sign 信息 | 来源数据与简化符号规则；与训练参数分开 |
| groups／roles | 名称到节点索引、输入／输出角色，保留配置出处 |
| meta | 数据版本、上游 commit、筛选阈值、缺失统计、子图范围、映射版本、许可 |

CSC 转边表时 coo.col 是 pre/src，coo.row 是 post/dst；不能按通常 adjacency 直觉反过来。GroupSpec 解析出的索引随子图变化，body_id 身份必须保持。

上游 NPZ 的 signed data 可提供绝对突触计数和有效符号，但不能恢复递质预测置信度、未知递质 provenance 或未保留的空间坐标；需要这些字段时须从原始注释另存，不能声称已有数据完全覆盖。

## 必须核对的上游建模选择

- 默认 min_synapses=3，过滤自连接、未知端点及部分无注释／非神经元条目；每次记录实际筛选效果。
- 递质符号表把多种调质递质简化成正号，未知项默认正号；保留 unknown 标记或单独报告，不能视为生理事实。
- sensorimotor_core 取输入下游与输出上游可达集合交集，再强制加回输入输出；因此“节点保留”不等于“路径连通”。
- max_neurons 会按子图列绝对权重和择取中间节点，可能切断通路；输入输出必须保留时还可能超过该数值。构建后重新做预算与可达性校验。
- 上游数据有循环，不允许为满足首版 DAG 静默删边；转入连续值动态核心的后续阶段。

## 依赖与许可

优先通过锁定版本的外部可选数据依赖调用上述纯数据函数，不把整个 FlyDrones 放进首版环境。安装层仍需核对上游包依赖与导入副作用；“不实例化 LIF”并非“上游包完全不导入 LIF 模块”。训练端加载中立图资产，不依赖 SDK／GUI／LIF。

MIT 代码保留版权与许可；MaleCNS 数据按上游 THIRD_PARTY.md 标注 CC-BY 4.0，实际使用时确认发布方版本和署名要求。D:\repository 仅是本机源码来源，不作为将来部署机器必需的绝对路径。

## 改动记录与验收

- **源码位置**：页末本地 FlyDrones 链接；项目离线适配尚不存在。
- **当前行为**：已有上游源码；未构建或验证本项目真实图资产。
- **计划操作**：复用加载／分组／提取，新增版本化资产和必要检查。
- **改后行为**：真实连接数据可独立进入 PyTorch Actor，不携带脉冲仿真执行链。
- **关联影响**：[[04 后续升级/07 感觉运动节点映射与可训练读出]]、[[04 后续升级/04 循环状态与 PPO 序列训练]]、模型保存／导出。
- **验收条件**：假数据方向检查、分组非空、往返身份一致、真实数据计数、子图可达性与循环报告。
- **审查状态**：后续未实施；仅源码静态核查。

## 已核查本地源码

- [src/flydrones/brain/connectome.py](<D:/repository/FlyDrones/src/flydrones/brain/connectome.py>)
- [src/flydrones/brain/brain.py](<D:/repository/FlyDrones/src/flydrones/brain/brain.py>)
- [src/flydrones/cli.py](<D:/repository/FlyDrones/src/flydrones/cli.py>)
- [src/flydrones/defaults.yaml](<D:/repository/FlyDrones/src/flydrones/defaults.yaml>)
- [tests/test_malecns_loader.py](<D:/repository/FlyDrones/tests/test_malecns_loader.py>)
- [tests/test_brain.py](<D:/repository/FlyDrones/tests/test_brain.py>)
- [LICENSE](<D:/repository/FlyDrones/LICENSE>)
- [THIRD_PARTY.md](<D:/repository/FlyDrones/THIRD_PARTY.md>)
