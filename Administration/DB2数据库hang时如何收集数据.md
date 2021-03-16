DBA在日常运维的过程中，或多或少会经历数据库hang住不能影响的问题。本文描述DB2数据库几种不同类型的hang住现象，及如何收集相应数据供后台专家分析问题根因。本文档仅适用于Unix或Linux 或Windows上的单分区数据库，如果是DB2 PureScale或者DB2 DPF集群环境，还需要收集集群相关数据。本文数据收集会尽量涵盖故障分析所需的完整的数据集，包括数据库lock、latch、thread stack和内存等状态，但有时也需要从OS 命令/API获取更多进行分析。

DB2数据库hang一般分以下情形：

# i.	Hard Hang：
此时DB2不能响应任何CLP命令（如：db2 "list applications"不返回任何信息或提示）。DB2通常会发出许多OS级别调用，这些调用需要在kernel space中执行，并且可能因各种故障原因卡住。从DB2用户角度来看，此时：发起新的DB2连接请求 或者 在现有DB2连接会话中执行任何SELECT或DML SQL都会被挂起。db2pd 命令有些情况下也会hang住并报timeout超时错误；有些情况下db2pd能工作，但别的命令如：db2 snapshot命令或 db2mon 监控不工作。
# ii.	Passive Hang：
DB2引擎有响应，但速度非常缓慢。而OS命令则运行良好，没有任何延迟。如：db2 "list applications"需要几分钟才能返回值。系统相关命令vmstat/iostat/top/topas命令显示没有任何问题。
 
  
    

# 数据收集:
在DB2 hang住状态下，分析问题的最基本数据是 db2sysc进程所有edus的stack traces。我们以两种方式收集stack数据，一种从DB2进程用户空间，另一种OS内核空间。在某些情况下，若stack dump信号位于内核代码路径中，signal handers可能会错过它。
因此第一要务是分清数据库hang属于哪一种情况，然后决定用哪个最常见的数据收集命令： db2fodc -hang full或db2fodc -hang basic。
# A.	db2fodc -hang full：
它使用db2cos（data collection script）收集数据，包括：stack、trace、snapshot和OS信息。它适合于Passive Hang的情况。
# B.	db2fodc -hang basic：
顾名思义，它是full option收集数据信息子集。它适合尝试新连接会hang住，但现有的连接还能处理查询和DML请求的情况。
# C.	Hard Hang时数据收集：
# C.1	当所有DB2 CLP命令包括db2pd都hang住的情况下，假如OS系统资源（特别是内存）并未耗尽，按下列方式进行数据收集：
# C.1.1	AIX系统:

vmstat -Iwt 1 30 > vmstat_Iwt.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

iostat -RDTVl 2 15 > iostat_RDTVl.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

ps -kelf >ps.kelf.\`date '+%Y-%m-%d-%H.%M.%S'\`

svmon -P \`db2pd -edus | grep "^db2sysc" | awk '{ print $3 }'\` > svmon.P.\`date '+%Y-%m-%d-%H.%M.%S'\`

svmon -G >svmon.G.\`date '+%Y-%m-%d-%H.%M.%S'\`

获取db2sysc 进程PID：ps -ef |grep db2sysc

ps -mo THREAD -p <db2sysc pid> >db2sysc.thread.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
procstack <db2sysc pid> >pstack.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
等待 30 秒

procstack <db2sysc pid> >pstack.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
等待 30 秒

procstack <db2sysc pid> >pstack.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
# 以root用户运行：

下载pdump.sh工具: https://www.ibm.com/support/pages/pdump-tool-overview  并运行

pdump.sh -l  <db2sysc pid>
    
sleep 60

pdump.sh -l <db2sysc pid>
    
注意：db2sysc可能有很多现成, dump可能会需要一些时间。

如果上述命令不工作，可尝试如下kdb命令或者所有threads信息：

echo 'th *' | kdb -script >thread.list

egrep -v "NONE|ZOMB" thread.list | egrep pvthr | tr '*!>' ' ' | awk '{print "f "$2}' | kdb -script > thread.stacks.\`date '+%Y-%m-%d-%H.%M.%S'\`

等待 30 秒

echo 'th *' | kdb -script >thread.list

egrep -v "NONE|ZOMB" thread.list | egrep pvthr | tr '*!>' ' ' | awk '{print "f "$2}' | kdb -script > thread.stacks.\`date '+%Y-%m-%d-%H.%M.%S'\`


# C.1.2	LINUX :-
vmstat 1 30 > vmstat.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

iostat -xt 2 15 > iostat_xt.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

ps -eLf >ps_eLf.\`date '+%Y-%m-%d-%H.%M.%S'\`

top -b -n 2 -d 10 >top.\`date '+%Y-%m-%d-%H.%M.%S'\`

获取db2sysc进程PID: ps -ef |grep db2sysc

# 以db2实例用户或root用户运行：

gstack <db2sysc pid> >gstack.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
等待 30 秒

gstack <db2sysc pid> >gstack.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
等待 30 秒

gstack <db2sysc pid> >gstack.\`date '+%Y-%m-%d-%H.%M.%S'\`
    
上面命令将获取用户态线程stack。

在root用户下用如下命令获取kernel态线程stack:

echo “1” >  /proc/sys/kernel/sysrq

echo “t” > /proc/sysrq-trigger  #它将dump所有线程stack

dmesg > kernel.stack.\`date '+%Y-%m-%d-%H.%M.%S'\`

等待 30 秒

echo “1” >  /proc/sys/kernel/sysrq

echo “t” > /proc/sysrq-trigger  #它将dump所有线程stack

dmesg > kernel.stack.\`date '+%Y-%m-%d-%H.%M.%S'\`

echo “0” > /proc/sys/kernel/sysrq


# C.1.3	Windows: -
db2bddbg -d db2ntDumpTid <path> -1 stack.1 
    
等待 30 秒

db2bddbg -d db2ntDumpTid <path> -1 stack.2
    
Once it is done try to format the stacks using

db2xprt stack.1 stack.fmt.1

db2xprt stack.2 stack.fmt.2



# C.2	当所有DB2 CLP命令hang住的情况下，但db2pd正常工作情况下，按下列方式进行数据收集：

# C.2.1	AIX:

# OS命令：

vmstat -Iwt 1 30 > vmstat_Iwt.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

iostat -RDTVl 2 15 > iostat_RDTVl.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

ps -kelf >ps.kelf.\`date '+%Y-%m-%d-%H.%M.%S'\`

svmon -P \`db2pd -edus | grep "^db2sysc" | awk '{ print $3 }'\` > svmon.P.\`date '+%Y-%m-%d-%H.%M.%S'\`

svmon -G >svmon.G.\`date '+%Y-%m-%d-%H.%M.%S'\`

# DB2命令：

db2pd -alldbs -mempool -memset -dbptnmem -inst >db2pd.mempool.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2pd -alldbs -active -apinfo -agent >db2pd_age.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2pd -edus -rep 30 3 >db2pd.edus.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2pd -stack all dumpdir=\`pwd\` -rep 30 3>stack.\`date '+%Y-%m-%d-%H.%M.%S'\`


# C.2.2	Linux:
# OS命令：

vmstat 1 30 > vmstat.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

iostat -xt 2 15 > iostat_xt.issue.\`date '+%Y-%m-%d-%H.%M.%S'\`

ps -eLf >ps_eLf.\`date '+%Y-%m-%d-%H.%M.%S'\`

top -b -n 2 -d 10 >top.\`date '+%Y-%m-%d-%H.%M.%S'\`

cat /proc/meminfo > meminfo.txt.\`date '+%Y-%m-%d-%H.%M.%S'\`

mpstat -P ALL 1 15 > mpstat.txt. .\`date '+%Y-%m-%d-%H.%M.%S'\`

pidstat -t -u -T TASK 1 15 > pidstat.txt. .\`date '+%Y-%m-%d-%H.%M.%S'\`
 
pidstat -r 1 15 > pidstat_mem.txt. .\`date '+%Y-%m-%d-%H.%M.%S'\`

pidstat -d 1 15 > pidstat_disk.txt. .\`date '+%Y-%m-%d-%H.%M.%S'\`

# DB2命令：

db2pd -alldbs -mempool -memset -dbptnmem -inst >db2pd.mempool.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2pd -alldbs -active -apinfo -agent >db2pd_age.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2pd -edus -rep 30 3 >db2pd.edus.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2pd -stack all dumpdir=\`pwd\` -rep 30 3>stack.\`date '+%Y-%m-%d-%H.%M.%S'\`


# C.2.3	Windows:-

# DB2/OS命令：
db2pd -winx >winx.out

db2pd -vmstat 1 10>vmstat.out

db2pd -alldbs -mempool -memset -dbptnmem >db2pd.mempool

db2pd -alldbs -active -apinfo -agent >db2pd_age

db2pd -edus -rep 30 3 >db2pd.edus

db2pd -stack all dumpdir=\`pwd\` -rep 30 3>stack.out

将在当前目录生成db2pd Stacks数据，通过db2xprt格式化它：

db2xprt <stack.bin> >stack.fmt.1


# C.2.4	针对所有平台收集：
db2trc on -l 512m  -t

等待 1 分钟

db2trc dump trace.dmp.\`date '+%Y-%m-%d-%H.%M.%S'\`

db2trc flw \`ls trace.dmp.* |tail -1\` trace.flw.\`date '+%Y-%m-%d-%H.%M.%S'\` -t

db2trc fmt \`ls trace.dmp.* |tail -1\`  trace.fmt.\`date '+%Y-%m-%d-%H.%M.%S'\`


参考链接: https://www.ibm.com/support/pages/db2-hang-data-collection

