# ICTRL 模块学习

> ICTRL 学习入口（MOC）。相关文档按 **SPEC/MAS 学习 → 代码级流程 → 验证环境** 分层组织，统一放在 `Shader模块学习/ICTRL/` 目录下。

## 文档导航

| 层级 | 文档 | 内容 |
| ---- | ---- | ---- |
| SPEC/MAS | [[ICTRL/01-IF 取指\|01-IF 取指]] | 取指行为、L0 cache 结构、指令存储、Cacheline FSM |
| SPEC/MAS | [[ICTRL/02-IS 调度\|02-IS 调度]] | 指令依赖检查、调度发射、Barrier workflow |
| SPEC/MAS | [[ICTRL/03-CLC\|03-CLC]] | Cluster Launch Control 整体流程与 WG ID 映射 |
| 代码级 | [[ICTRL/04-指令处理流程\|04-指令处理流程]] | RTL 模块层级、依赖检查信号、CC checker、发射流水 |
| 验证 | [[ICTRL/05-验证环境\|05-验证环境]] | 环境激励、资源管理、rm 分析、接口学习 |
| 验证 | [[ICTRL/06-问题记录\|06-问题记录]] | 学习过程中的疑问与待确认事项 |

相关模块：[[Shader模块学习/Valu 模块学习|VALU 模块学习]]

## 术语表

| 简写 | 英文全称 |
| ---- | ---- |
| IF | Instruction Fetch |
| IS | Instruction Scheduler |
| IB | Instruction Buffer |

## 整体工作流（MP）

1. Scheduler 将任务分配给 WM
2. WM 向 ICTRL 发送 wave init 信号和 fetch 请求
3. ICTRL-IF 取回指令数据，写入 ICTRL-IS 的指令缓冲区
4. ICTRL-IS 检查指令依赖关系，调度相应的 wave 发射指令
5. ICTRL-IS 从 ILO Cache 读取指令数据
6. ICTRL-IS 将指令数据发射至 VAC/NSR，并设置 BC
7. VAC 和 NSR 收集操作数，发送至目标流水线
8. 流水线等待源数据返回后执行
9. VAC、NSR 或流水线清除 BC

## CLC（Cluster Launch Control）

![[CLC整体流程.png]]

详见 [[ICTRL/03-CLC|03-CLC]]

## ICTRL 取指数据流

![[Pasted image 20261106145149.png]]
