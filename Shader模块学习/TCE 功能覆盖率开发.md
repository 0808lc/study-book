# 一、AI 分析流程
1、首先让AI 吃 ISA SPEC，撰写各个指令的域段信息与约束
2、根据约束让其生成功能覆盖率，同时记得引导AI采用标准的格式
3、让AI检查一下，是否存在一些隐含的约束，由多个约束串联起来构成

# 二、覆盖率分析 --11.06
## wmma-----修改完，重新收集覆盖率
1、src0/1_reg --R254/255，src0/1_scale_offset --R255，src2_dst_reg--R254/255未覆盖
![[Pasted image 20261030094611.png]]

2、k=16,src0_scale_reg_type=R,scale_block_size=16未覆盖----已经补充ignore_bins
 分析：k=16只会出现在A/B type为 FP16的情况，而src0_scale_reg_type=R要求A/B type为FP4/8
## SQMMA
### (1) src2_dst_reg_offset 未覆盖253\254\255---环境约束
![[Pasted image 20261102102342.png]]
![[Pasted image 20261102110157.png]]

### (2) src2_dst_reg_type 未覆盖 none---环境约束
 ![[Pasted image 20261102102502.png]]
![[Pasted image 20261102110350.png]]

### (3)src0_reg/,src0/1_shape,a/b_data_type部分随机未覆盖
src0_reg_type=C,src0_shape=K，src1_shape=N，a/b_data_type=BF16
![[Pasted image 20261104133314.png]]

### (4)m,n,op部分场景未覆盖----随机未覆盖-----优先级高
OP=ROW_MAX,M = 256,N = 256/240/224/208/192/176/160/144
![[Pasted image 20261104134406.png]]
### (5)reduce cross 部分场景未覆盖---部分环境约束，部分随机未覆盖 ------对齐

![[Pasted image 20261104134732.png]]
1、环境约束 ROW_MAX 中reduce_dst_type=R
![[Pasted image 20261104141714.png]]
### (6)op与reg_type_cross部分场景未覆盖---环境约束---对齐
![[Pasted image 20261102104142.png]]
![[Pasted image 20261102140042.png]]
### (7)src2_dst_reg 部分场景未覆盖--环境约束---对齐

![[Pasted image 20261102104239.png]]

## COMMON
### (1) dm ---环境约束
![[Pasted image 20261105104603.png]]
![[Pasted image 20261105104801.png]]

### (2) task_id  ----只采集了mp0_dmpe0的接口，后续补充
![[Pasted image 20261105104925.png]]
![[Pasted image 20261105105641.png]]

### (3) vr_size---环境约束 1-32-----搞vr_size =0场景（低）
![[Pasted image 20261105110442.png]]
![[Pasted image 20261105110328.png]]


# 三、覆盖率分析--11.10
### (1) src2_dst_reg_offset 未覆盖253\254\255---环境约束
![[Pasted image 20261102102342.png]]
![[Pasted image 20261102110157.png]]

### (2)cross a/b/cd/op----需要改造一下，将合法的bins删除，生成所有的bins
row_max 的一些混合数据类型未覆盖
![[Pasted image 20261110144519.png]]
**分析：ROW_MAX 要求reduce_dst_reg_type必须为R ???**  如果为R,那么cd 只能为F32
![[Pasted image 20261110145511.png]]
### (3)cross a/b/k/m/n/op row_max 随机太少，增加回归次数

![[Pasted image 20261110151939.png]]

### (4)这里row_max scale_reg_type=none 补充一下，解除SQMMA中ignore_bins，补充约束k=32,scale_reg_type!=R
![[Pasted image 20261110152739.png]]
![[Pasted image 20261110153623.png]]

### (5)补充ignore_bins,SQMMA/PAIR src0_reg!=C，ROW_MAX=C
![[Pasted image 20261110154119.png]]

### (6)row_max中m=256,n=256/240/224/208/192/176/160/144 ----环境约束
2**m *(n+1)*4 <= 256 ,m=3,(n+1)<=8,n=0~7
![[Pasted image 20261110154703.png]]

![[Pasted image 20261110154316.png]]
### (7)随机化不够
![[Pasted image 20261110160759.png]]
### (8) 补充Ignore 覆盖率 k=32,src0_scale_reg=none
![[Pasted image 20261110161212.png]]

### (9) 补充sqmma m!=256  SQMMA_PAIR m!=32
![[Pasted image 20261110161819.png]]
环境约束row_max dst_inner_stride  只能为none

![[Pasted image 20261110162403.png]]









# 四、覆盖率分析--11.12
###  (1) cp_src0_scale_reg_type == none的权重加大，row_max 权重调大
![[Pasted image 20261112102611.png]]


### (2) 环境约束
![[Pasted image 20261112113831.png]]
![[Pasted image 20261112114135.png]]
# 环境约束确认
## 1、row_max中环境中有如下约束reduce_dst_reg_type == R，dst_inner_stride == none  cd_data_type == FP32 src2_zero_init_type == none
![[Pasted image 20261110163619.png]]


# 覆盖率补充
swizzle cross a/b type m/n/k
![[Pasted image 20261112171552.png]]