---
title: hpc tutorial
date: 2020-11-14 15:37:41
tags:
---
### 注册
https://nic.csu.edu.cn/info/1146/1789.htm

### slurm简介
slurm是集群使用的作业调度系统,申请节点计算资源(cpu与gpu资源)并运行自己的程序需要编写shell脚本实现,即```*.sh```文件
### 重要的目录
1. Softwares:/public/software; # anaconda3等软件在这个目录下.
2. Job templates: /public/job_templates; # 样例脚本, 参考[信网中心提供的指南](https://nic.csu.edu.cn/info/1146/1790.htm)来使用样例脚本
### 常用命令
#### 查看集群所有节点的状态
>sinfo
```Bash
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST 
cpuQ*        up   infinite      3  down* node[0167,0445,0549] 
cpuQ*        up   infinite      2  drain node[0364,0391] 
cpuQ*        up   infinite      6    mix node[0069,0395,0625,0743,0874-0875] 
cpuQ*        up   infinite    989  alloc node[0001-0068,0070-0166,0168-0363,0365-0390,0392-0394,0396-0444,0446-0548,0550-0624,0626-0742,0744-0873,0876-1000] 
ResQ         up   infinite      1  drain node1009 
ResQ         up   infinite      1    mix node1008 
ResQ         up   infinite     13  alloc node[1002,1004-1007,1011-1012,1017-1022] 
ResQ         up   infinite      7   idle node[1001,1003,1010,1013-1016] 
gpu2Q        up   infinite      5    mix gpu[201,205-208] 
gpu2Q        up   infinite      5  alloc gpu[202-204,209-210] 
gpu4Q        up   infinite      1  down* gpu407 
gpu4Q        up   infinite      3    mix gpu[402,409-410] 
gpu4Q        up   infinite      8  alloc gpu[401,403-406,408,411-412] 
gpu8Q        up   infinite      4    mix gpu[801-804] 
fatQ         up   infinite      1    mix fat09 
fatQ         up   infinite      9  alloc fat[01-08,10]
```
cpuQ为cpu分区,gpu2Q~gpu8Q为gpu分区,如果想使用gpu,必须将作业提交到gpu分区
###### STATE
+ down
+ drain
+ mix
+ alloc: 
+ idle: 当前节点空闲
#### 查看自己提交的任务
>使用```squeue```查看```JOBID```

>使用```scontrol show job JOBID```追踪任务
#### 取消任务
>scancel JOBID
#### 更新任务
> scontrol update jobid=JOBID ...

由于可修改的属性非常多，我们可以借助 SLURM 自动补全功能来查看可修改的内容。 这只需要我们在输入完 JOBID 后空一格并敲两下\<TAB> 键。
```bash
account=<account>                      mintmpdisknode=<megabytes>             reqnodelist=<nodes>
conn-type=<type>                       name>                                  reqsockets=<count>
contiguous=<yes|no>                    name=<name>                            reqthreads=<count>
dependency=<dependency_list>           nice[=delta]                           requeue=<0|1>
eligibletime=yyyy-mm-dd                nodelist=<nodes>                       reservationname=<name>
excnodelist=<nodes>                    numcpus=<min_count[-max_count]         rotate=<yes|no>
features=<features>                    numnodes=<min_count[-max_count]>       shared=<yes|no>
geometry=<geo>                         numtasks=<count>                       starttime=yyyy-mm-dd
gres=<list>                            or                                     switches=<count>[@<max-time-to-wait>]
licenses=<name>                        partition=<name>                       timelimit=[d-]h:m:s
mincpusnode=<count>                    priority=<number>                      userid=<UID
minmemorycpu=<megabytes>               qos=<name>                             wckey=<key>
minmemorynode=<megabytes>              reqcores=<count>
```
#### 查看历史任务
>sacct
### 安装python环境
1. 推荐在当前用户文件夹下创建一个目录并将环境安装到该目录下 command: mkdir ~/envs
2. 使用conda创建一个新的虚拟环境 command: /public/software/anaconda3/bin/conda create --prefix /path/to/your/dir (e.g., ~/envs/)
3. 先用``` /public/software/anaconda3/bin/conda init```命令来初始化一下bash
4. 使用```source activate /path/to/your/env```来激活环境
### 编写一个测试脚本
1. 创建py文件 
>vim ~/hello.py
   ```python
   import torch
   print("cuda",torch.cuda.is_available())
   print("Hello world!")
   ```
2. 创建slurm脚本文件 
>vim ~/hello.sh
```Bash
#!/bin/bash
#SBATCH -J hello                #任务名字
#SBATCH -o hello_%j.out            #保存屏幕输出到这个文件
#SBATCH --error hello_%j.err            #保存屏幕输出到这个文件
#SBATCH -t 12:00:00             #最长运行时间, 此处为12小时
#SBATCH -N 1                    #申请节点数
#SBATCH --ntasks-per-node=1     #每个节点分配的任务数
#SBATCH --cpus-per-task=4       #每个任务分配的cpu数
#SBATCH -p gpu2Q                #提交到gpu分区才能申请gpu
#SBATCH -w gpu202               #指定gpu节点,不写则自动分配
#SBATCH --gres=gpu:1            #申请1块gpu
#SBATCH --mail-type=end                   #邮件通知类型start/end/failed, end表示作业结束时邮件通知, 可选项
#SBATCH --mail-user=jzystc@live.com       #邮件通知邮箱, 可选项
eval "$(/public/software/anaconda3/bin/conda shell.bash hook)"
conda activate /path/to/your/env      #激活环境 
# python ~/prototype/main.py -few 1 -prefix exp1 -form Pre-Train  #运行代码
python ~/hello.py
```
>通知邮件内容

Subject: ```Slurm Job_id=33653 Name=prototype Ended, Run time 00:22:18, COMPLETED, ExitCode 0```
From: ```SLURM workload manager <slurm@mu01.localdomain>```
1. 提交任务
> sbatch ~/hello.sh

4. 查看任务状态
>squeue
```bash
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON) 
33458      cpuQ prototyp hpc19471 PD       0:00      1 (None) 
```
>scontrol show job 33458
```Bash
JobId=33452 JobName=prototype
   UserId=hpc194711074(1293) GroupId=hpc(500) MCS_label=N/A
   Priority=666 Nice=0 Account=hpc194711074 QOS=normal
   JobState=PENDING Reason=None Dependency=(null)
   Requeue=1 Restarts=0 BatchFlag=1 Reboot=0 ExitCode=0:0
   RunTime=00:00:00 TimeLimit=00:01:00 TimeMin=N/A
   SubmitTime=2020-11-13T20:16:58 EligibleTime=2020-11-13T20:16:58
   AccrueTime=2020-11-13T20:16:58
   StartTime=Unknown EndTime=Unknown Deadline=N/A
   SuspendTime=None SecsPreSuspend=0 LastSchedEval=2020-11-13T20:16:58
   Partition=cpuQ AllocNode:Sid=ln01:86688
   ReqNodeList=(null) ExcNodeList=(null)
   NodeList=(null)
   NumNodes=1-1 NumCPUs=4 NumTasks=1 CPUs/Task=4 ReqB:S:C:T=0:0:*:*
   TRES=cpu=4,node=1,billing=4
   Socks/Node=* NtasksPerN:B:S:C=1:0:*:* CoreSpec=*
   MinCPUsNode=4 MinMemoryNode=0 MinTmpDiskNode=0
   Features=(null) DelayBoot=00:00:00
   OverSubscribe=OK Contiguous=0 Licenses=(null) Network=(null)
   Command=/public/home/hpc194711074/my_slurm.sh
   WorkDir=/public/home/hpc194711074/prototype
   StdErr=/public/home/hpc194711074/prototype/slurm.out
   StdIn=/dev/null
   StdOut=/public/home/hpc194711074/prototype/slurm.out
   Power=
   MailUser=(null) MailType=NONE
```
5. 查看保存的屏幕输出
> cat ~/hello.out
```
cuda True
hello world
```

### 注意事项
1. 如果python代码中有中文,需要在py文件的第一行加上以下三行代码中的任一行,作用是声明文件的编码格式
```python
#coding=utf-8
#coding:utf-8
#-*- coding:utf-8 -*-
```
2. 安装pytorch时,cuda版本需要低于10.2,否则可能报以下错误:```The NVIDIA driver on your system is too old (found version 10010).```
- [x] pytorch 1.7.0 + cuda 10.1 测试通过
- [ ] pytorch 1.7.0 + cuda 10.2 测试不通过
3. bash脚本的单行注释为#,sbatch参数也以#开头,注释sbatch参数的方法如下
   ```##SBATCH --job-name=xxx```
### 参考文献
[1] [北大工作站使用指南](http://bicmr.pku.edu.cn/~wenzw/pages/)
[2] [Slurm官方文档](https://slurm.schedmd.com/documentation.html)
[3] [中科大slurm使用指南.pdf](http://hmli.ustc.edu.cn/doc/userguide/slurm-userguide.pdf)
