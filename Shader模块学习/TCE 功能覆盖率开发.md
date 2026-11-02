# 一、AI 分析流程
1、首先让AI 吃 ISA SPEC，撰写各个指令的域段信息与约束
2、根据约束让其生成功能覆盖率，同时记得引导AI采用标准的格式
3、让AI检查一下，是否存在一些隐含的约束，由多个约束串联起来构成

# 二、覆盖率分析
## wmma
1、src0/1_reg --R254/255，src0/1_scale_offset --R255，src2_dst_reg--R255未覆盖
![[Pasted image 20261030094611.png]]

2、k=16,src0_scale_reg_type=R,scale_block_size=16未覆盖
 分析：k=16只会出现在A/B type为 FP16的情况，而src0_scale_reg_type=R要求A/B type为FP4/8