# 整体架构

VALU 由 **VAC、CMN MUX、4 个 VAG、ROUTER** 组成，负责 FOP、INT、BIT、CVT、FP64 指令的源操作数读取、计算与结果写回。指令以 wave32 封装，即 1 条指令包含 32 个 threads。

- **指令类型**：PVA 为 SIMD32，每周期 1 个 burst；SVA 为 SIMD16，每 2 周期 1 个 burst
- **VAC**：总体控制，负责指令顺序调度与源操作数读取，执行完成后向 BC 发送 barrier 解除信息
- **CMN MUX**：从 SHS / SLS 标量寄存器返回数据中为 src0、src1、src2 选择标量操作数
- **VAG**：一组 ALU 单元（8 PVA + 4 SVA），PRED / VR 返回数据直接接入计算；结果写回 PRED / VR，若 dst 为 SL 寄存器则经 ROUTER 从 instance0~31 找第一个有效 instance 写入

**每个VAG中有4个vadp，每个vadp中包含2个PVA和一个SVA，所以一共有32哥PVA和16个SVA，对应32个instance**
![[Pasted image 20260930143151.png|700]]

# valu验证环境
## 1、环境激励
环境的激励主要在valu_vseq中开始，在复位之后开始。
![[Pasted image 20261008102727.png]]<center></center>
### (1) rand_cfg_after_reset
 rand_cfg_after_reset( )主要用来初始化一些环境配置，根据不同的宏定义，初始化不同的值，直接赋值即可 **(TODO** --理解各个信号的含义)
 
### (2)m_mem_rw_seq.mon_mem_rw
m_mem_rw_seq.mon_mem_rw()则用来监测各个寄存器的mem相关的读写操作，包括vr\sl\shs\pred，这里一般是监测各个端口发出的请求，之后给出响应和数据，主要是**维护mem中的各个状态。**
<center></center><center></center> ![[Pasted image 20261008103755.png]]
 mem的状态通过一个mem_base_seq 进行维护，在创建时通过mem_init 进行初始化，mem中维护的资源如下，包含VR\SL\SHS\PRED以及vr/sl的实际地址
 ![[Pasted image 20261008104458.png]]

### (3) send_cache_invalidate
m_invalidate_seq.send_cache_invalidate负责发送缓存失效请求，并在valu空闲时刷新内存
+ 首先发送 mask=0的失效请求
+ 模拟真实场景中的时间间隔
+ 发送带mask的失效请求，指定wave失效，由于wave 最大为16个，因此mask也是16bit

### (4) m_pmpt_stall_seq
pmpt stallI 用来阻挡指令从instr fifo中弹出，达到压制功耗的目的，根据不同的实际场景进行仿真控制。
![[Pasted image 20261008111100.png]]

### (5) mon_excpt
mon_excpt主要用来监测exception的相关操作，记录端口中exception的种类并在final check中进行比较
![[Pasted image 20261008111355.png]]

### (6) wait_reset_jump_phase
用来监测是否跳转到复位，观察是否检测到复位事件的触发
![[Pasted image 20261008111755.png]]

### (7) insrt_timeout_mon
instr_timeout_mon用来监测指令是否超时

### (8)send_instr 

m_ictrl_seq.send_instr主要用来模拟ICTRL发送相关的指令，是环境激励中最主要的部分。
![[Pasted image 20261008133510.png]]
#### a、resource_table_init 
用来初始化每个wave的状态，包含wave_id\task_id以及各个资源的大小。
![[Pasted image 20261008113432.png]]

#### b、get_instr_num( )
get_instr_num()用来记录16个Wave中所有指令的数量
#### C、mon_instr_fifo_pop( )
mon_instr_fifo_pop用来监测指令fifo中指令的数量
#### d、decrease_wait_cycle()
decrease_wait_cycle 用来给每条指令的wait cycle减1 （**TODO** ：理解Wait cycle减一的规则 ）
#### e、manage_cp_count()
manage_cp_count 模拟硬件CP（command processor ）占用对发射的额外阻塞 （**TODO** ： CP资源是什么？在设计上如何理解？）
 1）累加规则：burst 期间（burst 周期），每发生一个周期的 wait_stall 事件（wait_stall.mask 位）即累加 cp_count，上限为 3（cp_count < 3 才累加，不会溢出）；
  但 fifo 满（fifo stall）时不会加，因为此时是发射阶段不能加。
 2）递减规则：当首指令 pc_offset 发生变化（即已发射进入下一条指令）且无 wait_stall   时，cp_count 自减，表示 CP 占用逐步释放。
3）作用：cp_count > 0 的 wave 在 get_next_wave_id 中会被剔除候选，从而模拟硬件上 CP 占用对后续指令发射的额外阻塞。
#### f、get_next_instr
get_next_instr用来获取下一条指令，首先清除空的wave，即所有指令发射完成的wave，之后获取下一个应该发射指令的wave_id，之后从该wave中获取指令进行发射。
![[Pasted image 20261008140131.png]]

get_next_wave_id的调度优先级如下(**正在发Burst的wave就一直发，其余随机**)：
该 task 负责从 resource_table 中选出下一个可发射的 wave，调度优先级如下：
1）收集 ready wave：遍历 resource_table，仅保留因当前 wait_cycle_env <= 1 的 wave
    （即等待周期已满足、可发射的 wave）。
2）剔除被阻塞的 wave：对每个 ready wave，若被 wait_stall.mask 置位（硬件 wait stall）
    或 cp_count > 0（CP 占用）则移出候选；但 burst 期间（burst_cnt != 0）不要受
    stall 影响，不会被剔除。
3）若只剩一个 wave：若当前 wave 正处于 burst（burst_cnt > 0）且其 wait_cycle_env
    <= 1，则直接锁定 cur_wave_id 继续发射，保证 burst 的连续性。
4）否则按概率换 wave：非 burst 时，若当前 wave 仍在候选队列中，则以 20% 概率
    （$urandom_range(100,1) <= 20）shuffle 后随机挑选其他 ready wave；若当前 wave 已
    不在候选队列，则必然 shuffle 换 wave。最终 wave_id = cur_wave_id。
#### g、send_one_instr
在instr_fifo不为满的时候发送下一条指令
![[Pasted image 20261008142402.png]]

### (9)指令的生成--ictrl_wave_seq.sv文件
+ 只有wave_valid有效，才生成对应的指令信息
+ 如果是直接用例的话，则直接解析其对应的指令
+ 之后根据不同的test_type以及alu_sel 生成不同的指令
+ reuse_previous_instr() 用来构造reuse寄存器的情况，随机复用上一条指令的寄存器地址
+ split_burst()用来将burst的指令进行拆分成为多个指令
+ reuse_check() 需要监测指令是否能够被reuse，如果dst覆盖该src，则不能reuse，设置reuse位
+ hazard_check() 为整个wave内每条指令计算发射等待周期wait_cycle
![[Pasted image 20261008144137.png]]

**指令生成流水线**
![[Pasted image 20261009141448.png]]
## 2、指令约束
ictrl_vac_issue_trans是ictrl发给valu的指令信息，每条指令的约束信息都在该Transaction中，每个人不同的指令都有不同的约束。
## 3、cmodel与dpi
1、首先，需要编译动态库dpi，在sim.cfg中的sim_opts中加上-sv_lib  +动态库的路径
2、环境中通过export "DPI-C" 和import "DPI-C" 与cmodel进行联合调试，主要文件在mp_dpi.sv与mp_dpi_define.sv中（**TODO** ：DPI不是调用函数嘛？为什么可以和cmodel进行联合调试嘛？cmodel内部会自己运行嘛？）
## 4、reference model学习

reference model主要作为参考模型，在valu的验证中，使用cmodel进行联合仿真，将RTL的输入信号通过dpi传递给cmodel，cmodel处理完成之后通过TLM接口写入到scoreboard，下面将详细的分析这个流程。
![[Pasted image 20261010111430.png]]
### (1) mon_ictrl_vac_instr
mon_ictrl_vac_instr主要接受发送的指令，之后将其转化为cmodel的数据类型
### (2)send_instr_to_model
当指令从ictrl_fifo中弹出时，同步调用dpi_tb2c_mp_ictrl_vac(data)函数，将指令信息发送给cmdel
### (3)wave_stall_check
用来检查wave_stall 功能是否正确
### (4)barrier_checker
检查读写bar清除接口发送的时间是否一致，读bar为固定延迟2个cycle，清除写bar则根据指令类型，分别固定不同的cycle。
### (5) invalid_checker
检查invalid相关功能，在Burst期间src不会被刷掉，busrt结束后如果法invalid，则对应的src不会被重用
## 5、DPI编译流程
下面路径下运行该命令，启动编译
![[Pasted image 20261010161241.png]]
![[Pasted image 20261010161218.png]]


# 相关疑问
## 1、为什么需要冒险检查这些，不能在transaction中进行约束嘛？
答：因为这是指令队列，transaction只生成单个指令，不对整个队列进行处理
## 2、cmodel如何返回数据给验证平台呢？
答：首先，cmodel调用dpi函数的时候，函数的参数是inout类型，这是参数引用类型，sv侧修改，c侧也能看见。同时，cmodel调用函数时，也会将数据封装成为事务，通过tlm接口传给scoreboard
## 3、valu中一个VR寄存器是如何映射到32个instance？
valu首先根据指令的逻辑地址，从VR中读出对应的物理地址，之后根据instance_active_mask去VR中读取数据，根据instance_active_mask，可以将32个instance划分为四个dblock，分别对应valu_vr0的四个接口，每个接口数据位宽256bit，供8个instance使用

![[Pasted image 20261019112825.png]]
![[Pasted image 20261019113955.png]]