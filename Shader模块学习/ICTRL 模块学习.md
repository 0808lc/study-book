# 一、ICTRL验证环境学习
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