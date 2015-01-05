self-heal代码分析
==================
self-heal一共有四种触发条件：
stale subvolume detected
打开文件时，如果设置有data-self-heal=open
lookup detected pending operations
lookup做完的时候，会自动进行
checksums of directory differ
read dir的时候，会做简单的check
subvolume came online
当建立连接时，发现有client_up了
 
之前提到了，在子卷文件的xattr上，都会记录文件的信息，self heal正是根据这些信息判断哪些子卷的文件是可靠的。
self heal的步骤
首先会对所有的文件进行加锁，首先加非阻塞锁，如果加锁失败，则会进行阻塞加锁
然后open所有up的子卷的文件
读取文件的xattr信息
将所有子卷上该文件的xattr信息组成pending_matrix
pending_matrix的每一行为该子卷记录的所有子卷的changelog信息
根据changelog可以确定副本的几种状态：
INNOCENT，changelog均为0，即不指责对方也不指责自己
FOOL，自己的changelog中，自己的不为0，
WISE，自己的changelog为0，对方的changelog不为0
IGNORANT，忽略的，即该副本的ChangeLog丢失。

下面比较重要：标记source节点
如果所有的节点都是INNOCENT的话
如果是修复meta_data，会选择lowest uid的节点作为修复的source节点
如果是修复data：如果有文件大小不为0，且有个两个子卷的文件大小不相当，判断为脑裂。如果所有的文件大小都为0，直接返回。如果有子卷的文件大小为0，选取其他大小不为0的节点作为修复节点。
 
如果有WISE的节点存在的话，则会计算wisdom，
计算方法：
当前子卷为WISE，wisdom标记为1，如果有其他子卷也为WISE，切该subvol上的的pending状态是在指责自己，则标记为0
如果所有的WISE状态的节点的wisdom都为0的话，则标记为脑裂
否则，选取wisdom最大的节点作为source节点
 
如果没有WISE节点存在：
计算子卷节点的witness，witness等于该节点记录的其他节点的changelog之和。
选取witness最大的节点作为source节点
 
如果上述步骤中未能选取到source节点，将所有成功的节点选取为source节点。
 
选取完source节点，然后就开始修复了~~
glusterFS提供了两种修复算法，full和diff
如果有子卷的文件大小为0的话，选取full的方法；如果文件的大小在page_size之内（这意味着整个文件可以在一个STACK_WIND内传输完，比diff具有更好的速度），也会选取full的方法。
下面分别讲一下full和diff的方法
full：基本就是从之前选取的source上读，读完以后写到其他的节点上
diff：首先checksum，在子卷上按block去计算md5，然后比较md5，如果不同的话，标记为需要写的。然后进行读和写
 
两个副本均为WISE时发生脑裂，那么在哪种场景下会产生脑裂呢？我们还是以冗余度为2的情况举一个简单的例子：某文件X的两个副本位于物理机A和物理机B上，在A和B上分别运行着进程a和进程b，a和b持续通过各自所在的物理机上的客户端对文件X进行不同的写操作。然后物理机A和B之间网络中断，因为AFR在一个副本的情况下仍能不中断上层应用，所以进程a和进程b仍会持续运行，但因为网络中断，文件X在A和B上的副本数据不再一致且都认为对方是异常的，当网络恢复时，两个副本互相“指责”，即出现了脑裂。当然这是脑裂发生的场景之一，有时候是有可能发生脑裂，而有时候是必然发生脑裂。脑裂，也是很多人关心的一个问题，不能一概而论。