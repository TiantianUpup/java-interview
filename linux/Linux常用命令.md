> 开发过程中难免会遇到使用Linux命令的场景，不经常使用总是容易忘记，好记性不如烂笔头

### 写在前面
本篇文章将从存储、内存、网络、CPU、vim、文本、其他、查找、性能监控与优化十个方面来介绍Linux常用命令，并且只会介绍命令的常用可选参数，更多参数使用命令  --help进行查看

### 存储
- **df - h**
  查看磁盘文件，可以查看文件的大小、使用率、挂载情况
  ```
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/sda3        49G   41G  8.3G  84% /
  devtmpfs        126G     0  126G   0% /dev
  tmpfs           126G  4.0K  126G   1% /dev/shm
  tmpfs           126G  491M  126G   1% /run
  tmpfs           126G     0  126G   0% /sys/fs/cgroup
  /dev/sdb        1.9T  2.2G  1.8T   1% /mnt/disk6
  /dev/sdg        1.9T  2.0G  1.8T   1% /mnt/disk4
  /dev/sde        1.9T  2.4G  1.8T   1% /mnt/disk5
  /dev/sdf        3.7T   18G  3.5T   1% /mnt/disk1
  /dev/sdd        3.7T  3.4G  3.5T   1% /mnt/disk2
  /dev/sdc        3.7T  3.6G  3.5T   1% /mnt/disk3
  /dev/sda1       497M  144M  354M  29% /boot
  tmpfs            26G     0   26G   0% /run/user/0
  ```
  -h 方便阅读方式显示

- **lsblk**
list block的缩写，列出系统上的所有的磁盘列表，包括分区情况，不包含使用率
  ```
  NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda      8:0    0   59G  0 disk 
  ├─sda1   8:1    0  500M  0 part /boot
  ├─sda2   8:2    0   10G  0 part [SWAP]
  └─sda3   8:3    0 48.5G  0 part /
  sdb      8:16   0  1.8T  0 disk /mnt/disk6
  sdc      8:32   0  3.7T  0 disk /mnt/disk3
  sdd      8:48   0  3.7T  0 disk /mnt/disk2
  sde      8:64   0  1.8T  0 disk /mnt/disk5
  sdf      8:80   0  3.7T  0 disk /mnt/disk1
  sdg      8:96   0  1.8T  0 disk /mnt/disk4
  ```

- **du**
disk usage的缩写， 进入指定目录统计子目录占用文件系统数据块（1024字节）的情况。若没有给出指定目录，则对当前目录进行统计
  ```
  16	./.oracle_jre_usage
  16	./.ssh
  0	./.ansible/tmp
  0	./.ansible/cp
  0	./.ansible
  0	./elasticsearch_data_dirs
  0	./spark-warehouse
  484	./csv
  0	./.beeline
  ```

Q&A：
Q：df和lsblk的区别
A：lsblk 查看的是block device，也就是逻辑磁盘大小
      df查看的是file system， 即文件系统层的磁盘大小

### 网络
- netstat => 用于显示与IP、TCP、UDP和ICMP协议相关的统计数据，一般用于检验本机各端口的网络连接情况

  常用组合 netstat -aux | grep xxx， 查看xxx进程占用的端口

### 内存
- free => 显示系统内存的使用情况，包括物理内存、交换内存(swap)和内核缓冲区内存
```
              total        used        free      shared  buff/cache   available
Mem:      131845504    13272912   115299004      343580     3273588   117385392
Swap:      10485756           0    10485756
```
**参数说明：**
- Mem => 内存的使用情况。
- Swap => 交换内存的使用情况
- total => 系统总的可用物理内存和交换空间大小。
- used => 已经被使用的物理内存和交换空间。
- free => 还有多少物理内存和交换空间可用使用。
- shared => 被共享使用的物理内存大小。
- buff/cache => 被 buffer 和 cache 使用的物理内存大小。
- available => 还可以被应用程序使用的物理内存大小。

**可选参数说明：**
- s => 搭配n使用，每隔n秒显示内存信息
- h => 以适于人类可读方式显示内存信息，会以GB、MB为单位显示内存的使用情况


### CPU
- **top**
查看各进程CPU使用率
  ```
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                                                                                  
  11412 root      20   0 4605524   1.9g  20260 S 111.3  0.7   1703:52 java                                                                                                                                                                                                     
  ```
  **参数说明：**
    - PID => 进程id
    - USER => 进程所有者
    - PR  => 进程优先级
    - NI => nice值。负值表示高优先级，正值表示低优先级
    - VIRT  => 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
    - RES  => 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
    - SHR  => 共享内存大小，单位kb
    - S => 进程状态
      - D => 不可中断的睡眠状态 
      - R => 运行 
      - S => 睡眠 
      - T => 跟踪/停止 
      - Z => 僵尸进程
    - %CPU  => 上次更新到现在的CPU时间占用百分比
    - %MEM  => 进程使用的物理内存百分比
    - TIME+ => 进程使用的CPU时间总计，单位1/100秒
    - COMMAND  => 进程名称（命令名/命令行）

  **可选参数说明：**
  - d => 指定每两次屏幕信息刷新之间的时间间隔
  - n => 循环显示的次数 

  假设想以5秒10次刷新各进程的CPU使用率可以使用命令：
  ```
  top -d 5 -n 10
  ```
  查看某个进程的CPU使用率可以搭配grep命令：
  ```
  top | grep pid
  ```

### vim文本编辑
vim是查看日志常用的命令之一。vim有三种常用模式，分别是：
- 命令模式
- 输入模式
- 底层命令模式

进入vim时为命令模式，通过a或者i进入输入模式，输入模式下可以对文本内容进行查找，esc退出输入模式。可以将底层命令模式看作是特殊的命令模式，因为底层命令模式只是执行保存和退出命令。**常用的命令如下：**
**查找：**
- /关键字：可以一直按n往后继续寻找关键字
- ?关键字：可以一直按n往前寻找关键字

**删除：**
- d 删除所有

**保存：**
- wq 保存退出
- q! 不保存强制退出

### 查找命令
- whereis
- locate
- find => 在指定路径下进行查找，会在磁盘上进行查找，查找效率较低
- which => 在path环境变量所规范下的路径进行查找

which默认是去查找path这个环境变量所规范的路径去查找的。

### 文本分析
- awk => 对文本中的内容进行提取在进行处理

  用法： awk '{pattern + action}' {filenames}
比如想提取文件的某一列可以使用如下命令：awk '{print $1}' filename
- grep => 查找某个文件中的关键字
- sed => 对文件进行增删改查，现在还没遇到，遇到在进行补充

### 其他
- ps => 显示进程号 
常用用法 ps -ef xxx 或ps aux
两者的区别只是显示的格式不一样，前者是标准格式，后者是BSD格式
想要查看某个应用的进程可搭配使用grep命令
- tail => 显示指定文件的末尾若干行  
**可选参数：**
  - n => 输出文件的尾部n行内容
  - f => 显示文件最新追加的内容

  比如查看日志的最后200行可使用命令：tail -n 200 xxx.log

### 性能监控与优化
- **iostat**
 I/O statistics的缩写，对系统整体的磁盘操作活动进行监视，不能针对某个进程
  ```
  avg-cpu:  %user   %nice %system %iowait  %steal   %idle
             2.93    0.00    1.04    0.02    0.00   96.01
  ```
  参数说明：
  - %user => CPU处在用户模式下的时间百分比
  - %nice => CPU处在带NICE值的用户模式下的时间百分比
  - %system => CPU处在系统模式下的时间百分比
  - %iowait => CPU等待输入输出完成时间的百分比
  - %steal => 管理程序维护另一个虚拟处理器时，虚拟CPU的无意识等待时间百分比
  - %idle => CPU空闲时间百分比
 
  %iowait和%idle是最需要关注的两个参数。如果%iowait比例较高，存在IO瓶颈，在分析并发任务时，则表示该任务是IO密集型任务。如果%idle较高，则表示CPU利用率比较低

  定时显示信息：
  ```
  iostat 2  3
  ```
  表示每隔2s刷新一次，每次刷新后显示3次

- **vmstat**
Virtual Meomory Statistics的缩写，对操作系统**整体的**虚拟内存、进程、CPU活动进行监控，无法监控某个进程
  ```
   procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
   r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
   1  0      0 315549792   5552 78702144    0    0     1    52    4    4  4  2 95  0  0
  ```
  参数说明：
  - Procs
    - r => 运行队列中进程数量
    - b =>  等待IO的进程数量
  - Memory
    - swpd => 使用虚拟内存大小
    - free => 可用内存大小
    - buff => 用作缓冲的内存大小
    - cache => 用作缓存的内存大小
  - Swap
    - si => 每秒从交换区写到内存的大小
    - so => 每秒写入交换区的内存大小
  - IO
    - bi => 每秒读取的块数
    - bo => 每秒写入的块数
  - system
    - in => interrupt的缩写，每秒中断数，包括时钟中断
    - cs => count/second的缩写，每秒上下文切换数
  - CPU
    - us => 用户进程执行时间
    - sy => 系统进程执行时间
    - id => 中央处理器的空闲时间 
    - wa => 等待IO时间

  cs和wa、id是比较重要的参数，如果cs的值比较大，则吞吐量很低

- **mpstat**
查看所有CPU的信息或指定CPU的信息
  ```
  11:09:41 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
  11:09:41 AM  all    3.52    0.00    1.65    0.05    0.00    0.02    0.00    0.00    0.00   94.77
  ```
  参数说明：
  - %user => 表示处理用户进程所使用CPU的百分比
  - %nice => 表示使用nice命令对进程进行降级时CPU的百分比
  - %system => 表示内核进程使用的CPU百分比
  - %iowait => 表示等待进行I/O所使用的CPU时间百分比
  - %irq => 表示用于处理系统中断的CPU百分比
  - %soft => 表示用于软件中断的CPU百分比
  - %steal  => 显示虚拟机管理器在服务另一个虚拟处理器时虚拟CPU处在非自愿等待下花费时间的百分比
  - %guest  =>显示运行虚拟处理器时CPU花费时间的百分比
  - %idle => 显示CPU的空闲时间
  - %intr/s => 显示每秒CPU接收的中断总数
 
  %iowait和%idle是最需要关注的两个参数

  语法：
  ```
  mpstat -P CPU 时间间隔 采集次数
  ```
  比如每隔1s对第一颗CPU统计一次，并且每次打印五条：
  ```
  mpstat -P 0 1 5 
  ```

