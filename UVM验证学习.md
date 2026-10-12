# 一、基础知识学习
+ 队列随机----queue.shuffle()将数组、队列的随机元素打乱
+ uvm_wait_for_nba_region等待nba区域的内容被执行完成
+ \$value\$plusargs 函数---用来从run-option中提取参数的值
+ \$test\$plusargs 函数-- 用来从run-option中监测是否有这个值，通常用来区分仿真的各种情况、
+ 使用deposit 可以给信号赋予一个初始值
# 二、基础函数的用法
## 1、文件处理函数

| 函数名                           | 介绍                                                  | 示例                            |
| ----------------------------- | --------------------------------------------------- | ----------------------------- |
| $sscanf(str,format,args)      | $sccanf的三个参数，第一个是扫描对象，第二个是扫描格式，第三个是提取出来的参数          | $sscanf(str, "pc: %h", value) |
| <br>$sformat(str,format,args) | $sformat则是将参数args填到format中（按照格式要求），然后将format赋值给str。 |                               |

# 三、项目环境配置
## 1、module load project/hs_gpu_test
如下图所示，module show 对应的模块，可以查看配置文件的路径，也可以查看对应的配置
![[Pasted image 20261012153706.png]]