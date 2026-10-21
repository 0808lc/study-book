# 一、验证目标
比较cmodel与rtl 中的perf counter是否一致

**数据来源：**
![[Pasted image 20261019102629.png]]

# 二、验证方法

1. 根据perf_counters_define的csv文件中，提取**CSIM列为True**的perf counter与其对应的rtl path
2. 每个cycle对于该perf counter进行累加，仿真结束则与cmodel提供的值进行比较（perf_counter.csv文件）。
3. 最终输出perf_counter_result.txt文件
![[Pasted image 20261019103123.png]]

# 三、错误场景总结

1、表格上RTL_PATH填写错误，导致验证环境中抓取的信号错误（由于RTL_PATH直接复制粘贴导致）
2、RTL中perf counter 信号统计的不是次数，而是状态（统计1次，信号拉高多个周期）
3、一些dmkill/oob/一些弃用的指令，rtl和cmodel的统计不一致
4、RTL 读VR/SL，应该只读一次，却读了两次
5、部分perf counter   cmodel和RTL实现难以保持一致，CSIM列改为FALSE

# 四、未测试到的PFM
## ICTRL

| ICTRL_INSTR_DROP_CTRL                                                                                                                                                                                               | 正常不会跑到        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| ICTRL_INSTR_ILLEGAL                                                                                                                                                                                                 | 正常不会跑到        |
| ICTRL_NSR_ISSUE_CTRL_MUTEX                                                                                                                                                                                          | Csim not      |
| ICTRL_NSR_ISSUE_CTRL_OLCHK                                                                                                                                                                                          | 图形            |
| ICTRL_NSR_ISSUE_TCE_SQMMA                                                                                                                                                                                           | BT上QMMA还未测试   |
| ICTRL_ISSUE_INSTR_NUM_AS<br>ICTRL_ISSUE_INSTR_NUM_DS<br>ICTRL_ISSUE_INSTR_NUM_GS<br>ICTRL_ISSUE_INSTR_NUM_HS<br>ICTRL_ISSUE_INSTR_NUM_MS<br>ICTRL_ISSUE_INSTR_NUM_PS<br>ICTRL_ISSUE_INSTR_NUM_VS                    | HS中只有CCE一个DM  |
| ICTRL_NSR_ISSUE_IMAGE<br>ICTRL_NSR_ISSUE_ITP<br>ICTRL_NSR_ISSUE_PSO_DS<br>ICTRL_NSR_ISSUE_PSO_RD<br>ICTRL_NSR_ISSUE_PSO_WR<br>ICTRL_NSR_ISSUE_RTOP<br>ICTRL_NSR_ISSUE_SMP                <br>ICTRL_NSR_ISSUE_BUFFER | 图形相关以及一些弃用的指令 |

| ICTRL_NSR_ISSUE_CTRL_BAR | 增加ctrl激励，可以覆盖 |
| ------------------------ | ------------- |
## LMA
rtg中约束 atomic 8/128 是不支持的嘛？
![[Pasted image 20261020173426.png]]
![[Pasted image 20261020173039.png]]

| NSR0_LMA_LOCAL_ATOMIC128_REQ                                                                                                                                                                                                   | TODO |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---- |
| NSR0_LMA_LOCAL_ATOMIC8_REQ                                                                                                                                                                                                     | TODO |
| NSR1_LMA_LOCAL_ATOMIC128_REQ                                                                                                                                                                                                   | TODO |
| NSR1_LMA_LOCAL_ATOMIC8_REQ                                                                                                                                                                                                     | TODO |
| LMA_VTX_ACTIVE<br>LMA_LMS_VTX_RD_DATA_NUM<br>LMA_LMS_VTX_WR_DATA_NUM<br>NSR0_LMA_VTX_RD_REQ<br>NSR0_LMA_VTX_REQ<br>NSR0_LMA_VTX_WR_REQ<br>NSR1_LMA_VTX_RD_REQ<br>NSR1_LMA_VTX_REQ<br>NSR1_LMA_VTX_WR_REQ<br>VCC_LMA_ACTIVE<br> | 图形相关 |

## LSU

| LSU_L1_ATOMIC_REQ_GTCC<br>LSU_L1_ATOMIC_REQ_GTCD<br>LSU_L1_ATOMIC_REQ_GTCV<br>LSU_L1_ATOMIC_REQ_PDM<br>LSU_L1_RD_REQ_GTCC<br>LSU_L1_RD_REQ_GTCD<br>LSU_L1_RD_REQ_GTCV<br>LSU_L1_RD_REQ_PDM<br>LSU_L1_WR_REQ_GTCC<br>LSU_L1_WR_REQ_GTCD<br>LSU_L1_WR_REQ_GTCV<br>LSU_L1_WR_REQ_PDM | HS中DM只有CCE |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |

## SCHEDULE

| SCHED_TASK_RECEIVED_GTCC<br>SCHED_TASK_RECEIVED_GTCD<br>SCHED_TASK_RECEIVED_GTCV<br>SCHED_TASK_RECEIVED_PDM | HS中DM只有CCE |
| ----------------------------------------------------------------------------------------------------------- | ---------- |

## TCE

| MP_ACTIVE_CYCLE_TCE_ALU1<br>MP_ACTIVE_CYCLE_TCE_ALU2<br>MP_ACTIVE_CYCLE_TCE_ALU3                                                                                                                                                                                                                                                                                                        |           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| MP_TCE_ACTIVE_CYCLE_ROW_MAX                                                                                                                                                                                                                                                                                                                                                             | SQMMA相关待测 |
| MP_TCE_ACTIVE_CYCLE_WMMA_FP8_ALU                                                                                                                                                                                                                                                                                                                                                        |           |
| MP_TCE_ROW_MAX_INST                                                                                                                                                                                                                                                                                                                                                                     |           |
| MP_TCE_ACTIVE_CYCLE_SQMMA_BF16<br>MP_TCE_ACTIVE_CYCLE_SQMMA_FP16<br>MP_TCE_ACTIVE_CYCLE_SQMMA_FP4<br>MP_TCE_ACTIVE_CYCLE_SQMMA_FP6<br>MP_TCE_ACTIVE_CYCLE_SQMMA_FP8<br>MP_TCE_ACTIVE_CYCLE_SQMMA_S8<br>MP_TCE_ACTIVE_CYCLE_SQMMA_TF32<br>MP_TCE_ACTIVE_CYCLE_SQMMA_U8<br>MP_TCE_SQMMA_INST_HMMA<br>MP_TCE_SQMMA_INST_IMMA<br>MP_TCE_SQMMA_PAIR_INST_HMMA<br>MP_TCE_SQMMA_PAIR_INST_IMMA | SQMMA相关待测 |

## TME
笔误？
![[Pasted image 20261021093842.png]]
![[Pasted image 20261021095012.png]]

| MP_TME_GATHER4_INST<br>                                                                                                                                                                                          | TODO |     |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | --- |
| MP_TME_GATHERN_INST<br>                                                                                                                                                                                          |      |     |
| MP_TME_LD_BURST0_PORT0<br>MP_TME_LD_BURST0_PORT1<br>MP_TME_LD_BURST1_PORT0<br>MP_TME_LD_BURST1_PORT1<br>MP_TME_LD_BURST2_PORT0<br>MP_TME_LD_BURST2_PORT1<br>MP_TME_LD_BURST3_PORT0<br>MP_TME_LD_BURST3_PORT1<br> |      |     |
| MP_TME_LD_CMD_PORT0<br>MP_TME_LD_CMD_PORT1<br>MP_TME_LD_INST<br>                                                                                                                                                 |      |     |
| MP_TME_RTN_DATA_32B_PORT0<br>MP_TME_RTN_DATA_32B_PORT1<br>MP_TME_RTN_DATA_64B_PORT0<br>MP_TME_RTN_DATA_64B_PORT1<br>                                                                                             |      |     |
| MP_TME_ST_BURST0_PORT1<br>MP_TME_ST_BURST1_PORT1<br>MP_TME_ST_BURST2_PORT1                                                                                                                                       |      |     |

## valu

| VALU_ACTIVE_FOR_FRAG<br>VALU_ACTIVE_FOR_GEOM | 图形相关 |
| -------------------------------------------- | ---- |

## WM

| WM_WAVE_LAUNCHED_AMPLIFICATION<br>WM_WAVE_LAUNCHED_DOMAIN<br>WM_WAVE_LAUNCHED_FRAGMENT<br>WM_WAVE_LAUNCHED_GEOMETRY<br>WM_WAVE_LAUNCHED_HULL<br>WM_WAVE_LAUNCHED_MESH<br>WM_WAVE_LAUNCHED_VERTEX | HS中DM只有CCE |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |

## HS中删除

| RTU                                                                                                                                                                                                                                                                                                                                                                                                                                              | SMP                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| MP_RTU_RT_INSTRUCTION_NUM<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_AB<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_AH<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_AS<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_CM<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_CS<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_DR<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_DS<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_IX<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_PR<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_ST<br>MP_RTU_RT_INSTRUCTION_NUM_RTOP_TR | NSR_BUFFER_ACTIVE<br>NSR_IMAGE_ACTIVE<br>NSR_SMP_ACTIVE<br>NSR_SMP_THREAD_NUM |
