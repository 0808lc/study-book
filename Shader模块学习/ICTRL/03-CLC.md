# CLC 整体流程

> CLC（Cluster Launch Control，集群启动控制）让正在运行的 workgroup cluster 接管尚未下发 cluster 的工作，减少重复启动开销并改善负载不均衡。
> 上级：[[ICTRL/00-ICTRL 模块学习|ICTRL 模块学习]]

![[CLC整体流程.png]]

## 背景与范围

普通调度通常为输出 tile 创建相应的 workgroup，由硬件逐个下发。持久化调度用少量长期运行的工作组循环处理任务，减少启动与初始化开销，但静态分配可能造成执行长尾。CLC 保留按任务数量描述 grid 的方式，并允许现有 worker 动态接管尚未下发的 cluster。

取消的是目标 cluster 的正常启动/下发，其工作仍由接管者执行。HS 截图将动态接管限制在同一个 MPC 内，跨 MPC 的分配仍依赖静态策略与 multicore 配置。

## 首次任务下发

1. CCE 通过 BIF 读取控制流，将 kernel 展开为 workgroup cluster，再展开为 workgroup。
2. 仲裁逻辑选择 workgroup 发送至 MTC。MTC 构造 workitem/MP task、分配并初始化资源，把任务交给 MP。
3. cluster enable、rank、局部坐标等信息随 task/wave 传入 Scheduler，存储在 Sched-TS。
4. MP 中的 wave 按初始 WG ID 执行任务。`SR[75]`、`SR[76]`、`SR[77]` 分别提供 3 bit 有效的 cluster-local X/Y/Z 坐标，高位补零。

## 循环接管任务

1. **处理当前任务。** 计算体在截图 MT example 中省略；结合前文补入此步骤才能构成执行新任务的完整循环。具体请求是否与计算重叠，取决于实际程序。
2. **请求前同步。** cluster 内所有 wave 成对执行 `WGC_ARRIVE` 和 `WGC_WAIT`，使参与者进入一致的阶段。
3. **发起请求。** 按架构文字，仅第一个 workgroup 的第一个 wave 发出 `CLC.TRY_CANCEL`。请求由 MP ICTRL 经 Scheduler 传至 CCE。
4. **CCE 选择目标。** 若有尚未开始下发的 cluster，CCE 取消其正常下发，返回成功及该 cluster 的首 WG 全局坐标；随后跳过它，继续下发其他 cluster。否则返回失败。结果存于 Sched-TS。
5. **请求后同步。** 所有 wave 再执行一组 `WGC_ARRIVE/WGC_WAIT`。MT example 将其安排在 TRY_CANCEL 与 QUERY_CANCEL 之间，用于取消结果使用前的同步。
6. **所有 wave 查询。** `CLC.QUERY_CANCEL` 将状态写入 `SL_REG`。成功为 1，失败为 0。
7. **成功分支。** 按最新 SUBOP/MT example 描述，硬件自动更新各 wave 的 task-store WG ID。当前 cluster 回到计算体处理新任务，再重复该循环。
8. **失败分支。** 按本 HS 文档的条件，表示 CCE 无尚未下发的 cluster。当前 worker 不再接管任务，完成当前应做的工作并结束；不代表其他正在运行的 cluster 都已结束。

## WG ID 映射与调试

```text
新 WG ID.x = 被取消 cluster 的首 WG ID.x + cluster_local_x
新 WG ID.y = 被取消 cluster 的首 WG ID.y + cluster_local_y
新 WG ID.z = 被取消 cluster 的首 WG ID.z + cluster_local_z
```

例如首 WG 坐标为 `(8,4,0)`，当前 WG 局部坐标为 `(1,2,0)`，则接管后全局坐标是 `(9,6,0)`。每个 WG 保持其 cluster 内的相对位置；同一个 WG 的 waves 应得到一致的 WG ID。

`GET_WG_ID` 成功时读取被取消 cluster 的首 WG 坐标，结果写入 SL_REG，供调试或兼容用途。自动更新流程中，软件不必再次计算并写回。早期截图出现过 `GET_FIRST_WG_ID` 名称及软件计算、`UPD_WG_ID` 更新方案；不能将它与后来的自动更新叠加成必经步骤。`UPD_WG_ID` 的独立适用范围需结合最终 ISA 确认。

## ARRIVE / WAIT 的硬件细节

`WGC_ARRIVE` 发往 CTRL 流水线，使本地 `wave_wgc_flag` 从 ready 变成 arrived。到达消息经 Sched 的 Collect_WGC 传给 CCE-BarrierAsync，携带 task/wave 标识及 wgc_id。该 wave 可以继续发普通指令，但下一轮 ARRIVE 必须等上一轮同步完成。

`WGC_WAIT` 在尚未完成时将 arrived 变成 waited，并关闭 `issue_enable`。停用原因是截图中的 `WAIT_WGC_ARRIVE`（原名 `WAIT_WGC_BAR`）。WAIT 本身不替代 ARRIVE 的上报作用。

CCE-BarrierAsync 收齐本轮到达后，发出完成/重新激活通知。后续 64 bit mask 被拆分广播至各 MP ICTRL；位与 wave 的精确对应关系需看实现。

| 通知与 WAIT 的顺序 | 行为 |
|---|---|
| WAIT 先到、完成通知后到 | wave 已阻塞，收到通知后重新激活 |
| 完成通知先到、WAIT 后到 | 置 arrived_reactivated 保存完成状态；WAIT 到来后直接通过并清除记录 |
| 完成通知已到、wave 结束前未执行 WAIT | 在 wave 真正结束时清理残留记录 |

## 需要核对的边界

- MT example 写“first thread”，架构文字写“第一个 WG 的第一个 warp/wave”；最终发射粒度应以 ISA/RTL 为准。
- 请求后同步的程序意图是保证查询时可以使用取消结果。但 WGC 微架构截图只明确描述 arrive 计数与重新激活，没有给出 cancel completion 与 barrier 的具体关联条件，不能单凭“所有 wave 到达”证明异步请求完成。
- 自动更新 WG ID 的确切生效点、跨 wave 一致性，以及 QUERY_CANCEL 完成/后续计算之间的依赖条件，需看最终协议。
- 前文规则禁止 TRY_CANCEL、WGC_ARRIVE、WGC_WAIT 携带 read_bar/write_bar；这不改变 WGC 自身的同步语义。read_bar/write_bar 的具体依赖机制不能仅凭名称推定为内存屏障。

## 来源定位

本次对话中的截图章节：CLC Background / Use Case；CCE、MTC、MP 架构说明；取消成功与失败流程；2.3.2 CTRL.CLC_SUBOP；2.5.2 MT example；WGC_ARRIVE 与 3.1.2 CLC.WGC_WAIT。

图中“执行当前任务”是为了串联完整循环而补充的解释节点，其余主流程以这些截图为依据。图像源可由 `outputs/clc-flow-2026-11-03/render_clc.py` 重新生成。
