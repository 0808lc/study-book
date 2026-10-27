# 一、SPEC /MAS 学习

## 0、术语表

| 简写 | 英文全称 |
| ---- | ---- |
| IF | Instruction Fetch |
| IS | Instruction Scheduler |
| IB | Instruction Buffer |

## 1、工作流
MP 中 ICTRL 流水线的工作流程：
1. Scheduler 将任务分配给 WM
2. WM 向 ICTRL 发送 wave init 信号和 fetch 请求
3. ICTRL-IF 取回指令数据，写入 ICTRL-IS 的指令缓冲区
4. ICTRL-IS 检查指令依赖关系，调度相应的 wave 发射指令
5. ICTRL-IS 从 ILO Cache 读取指令数据
6. ICTRL-IS 将指令数据发射至 VAC/NSR，并设置 BC
7. VAC 和 NSR 收集操作数，发送至目标流水线
8. 流水线等待源数据返回后执行
9. VAC、NSR 或流水线清除 BC

## 2、IF (Instruction Fetch)

IF 位于 WM、L1I Cache 和 IS（Instruction Scheduler）之间，接收来自 WM 的单一 PC 流，从自身内部 cache（L0）或 L1I Cache 接口取指令。
### (1) 基本取指行为
- 对每个提交的 PC 值进行 cache 查找
- Fetch PC 按 4 dword 对齐
- Fetch 与 prefetch 各请求 2 次 cache 查找
- 每周期取 2 条指令
- Miss 请求不重新 fetch
- 记录 fetch 与 prefetch 的 pending 状态
- 分支指令检查，并 stall WM 的 fetch
### (2) L0 Cache 结构
- 维护两个子 cache，分别存放奇数 / 偶数 cacheline，均采用 **PLRU** 替换算法
- Cacheline 宽度 **64 Byte**
- Cache 锁机制：cacheline 在成功清零之前不会被丢弃
### (3) 指令存储
- 指令数据拆分为 Ctrl words 与 Data words，分别写入指令 ctrl buffer 和指令 store memory
- 指令 ctrl buffer：4 bank × 64 × 32 bit
- 指令 store memory：4 bank × 64 × 128 bit
### (4) L1I 交互与预译码
- Cache miss 时向外部 L1 指令 cache 发起请求
- L1I 指令数据返回时，对 src/dst range 进行预译码（pre-decode）
### (5) Invalidate
- 支持来自 CR 或 DFU 的 icache invalidate 操作
- 支持对所有 ctxt 或任意 ctxt 的 invalidate
### (6) Cacheline FSM
Cacheline 共有 **7 种状态**，编码如下：

| 状态 | 编码 | 含义 |
| ---- | ---- | ---- |
| `invalid` | 000 | 空 cacheline |
| `fetched` | 001 | fetch 读请求 miss，等待数据返回 |
| `valid` | 010 | 包含有效的指令数据 |
| `prefetched` | 011 | prefetch 读请求 miss，等待数据返回 |
| `invalid_fetched` | 100 | 被 invalidate 的 fetched cacheline |
| `invalid_prefetched` | 101 | 被 invalidate 的 prefetched cacheline |
| `invalid_valid` | 110 | 被 invalidate 的 valid 且 lcl≠0 的 cacheline |
## 3、IS (Instruction Scheduler)

IS 负责指令发射与 wave 管理，功能包括：
**信息存储与依赖检查**
1. 保存每个 wave 的信息和指令控制字（control words）
2. 检查指令依赖关系
3. 指令 CC 字段处理
**特殊流程处理**
4. Instance 全零处理
5. 指令 End 处理
6. Dm-killing 处理
7. Ctrl 指令处理
8. Wait-bar 指令在依赖 ready 之前提前进入 NSR
9. 向量寄存器重配置（Vector register reconfig）
10. 调试模式 Trap 支持
**调度与发射**
11. 调度 wave 发射指令（Schedule wave for instruction issue）
12. 维护 wave 发射状态机（wave issue FSM）
13. Fop 指令 burst 拆分
14. 提取 wave PC 采样信息
15. Sqmma 发射控制
**流控（Throttle）**
16. 长延迟指令节流（Long latency instr throttle）
17. 每 wave 在途指令数量节流（Per-wave inflight instr throttle）
### (1) Barrier workflow
1. instruction select, set barrier counter based on the read barrier and write barrier（指令选择时，根据读屏障和写屏障设置 barrier counter）
2. instruction issue to VAC/NSR（指令发射至 VAC/NSR）
3. VAC or NSR read all src done, clear read barrier（VAC 或 NSR 读完所有源操作数后，清除读屏障）
4. pipeline executes done, clear write barrier（流水线执行完成后，清除写屏障）
5. CB collects vr/sl/shs/pred write back and clears write barrier（CB 收集 vr/sl/shs/pred 写回并清除写屏障）
6. barrier counter broadcast barrier status to IS, per-wave's wait_mask snoop bc status and release dependency relationship（barrier counter 向 IS 广播屏障状态，每个 wave 的 wait_mask 侦听 bc 状态并解除依赖关系）


# 二、ICTRL验证环境学习

## 1、环境激励
ICTRL中的激励从ictrl_vseq开始的body开始，body内部有ictrl_job_gen()，drive_config()，ictrl_mon()，l1icache_init()，bc_process()，ictrl_job__send()这六个任务构成。
![[Pasted image 20261019135029.png|540]]
### (1) 资源管理
ICTRL中资源管理的力度和BT上基本一样，最基础的是task，因此是从task_info_base->shader_info_base->ictrl_workload_base这样开始
### (2) ictrl_job_gen
ictrl_job_gen( )主要用来初始化相关的参数
![[Pasted image 20261020101154.png]]

### (3) drive_config
主要用来初始化一些配置信号

### (4) ictrl_mon
主要用来在ictrl中做一些monitor

### (5)l1icache_init

l1icache_init主要可以分为三步，首先加载kill_shader.txt中的kill_shader的指令，之后加载Breakpoint.txt中breadpoint的指令，最后，初始化每一个shader的end_pc和start_pc 。
