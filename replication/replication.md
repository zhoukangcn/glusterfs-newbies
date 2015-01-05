冗余代码分析
==============
冗余技术（redundancy）是分布式存储系统的一个重要技术手段。最简单、最传统的冗余存储技术是副本技术（replication）：通过拷贝多个原始数据，并将它们分别存储在不同节点上，从而增加数据可靠性。Gluster文件系统通过多分冗余来保持其系统文件的高可用性。
基本流程

上图描述了复制卷文件交互的基本过程，基本上所有的操作都遵循如下的步骤：lock->pre_op->op->post_op->unlock。
读写文件
GlusterFS读写文件时的流程稍有不同：写文件时会将文件写到多个brick节点上；而读文件时只会选取一个节点进行读取。

AFR transaction type
一共有四种：
typedef enum {
        AFR_DATA_TRANSACTION,          /* truncate, write, ... */
        AFR_METADATA_TRANSACTION,      /* chmod, chown, ... */
        AFR_ENTRY_TRANSACTION,         /* create, rmdir, ... */
        AFR_ENTRY_RENAME_TRANSACTION,  /* rename */
} afr_transaction_type;
文件锁
同linux文件系统一样，GlusterFS需要文件锁，防止多个client对文件同时进行操作。
在进行写操作时会进行加锁的操作，加锁之前会指定lock_owner。其提供了两种加锁方式：阻塞式加锁和非阻塞式加锁，根据不同的请求类型选择不同的锁。
AFR_DATA_TRANSACTION & AFR_METADATA_TRANSACTION 非阻塞的inode锁
AFR_ENTRY_RENAME_TRANSACTION 阻塞的rename锁
AFR_ENTRY_TRANSACTION 非阻塞的目录锁
非阻塞的加锁时，可能会出现多个client同时请求文件锁的情况，这时该如何处理？inode锁和目录锁的处理方法不同
nonblock_inodelk的锁会同时向所有的child_up的client发起加锁的请求，一旦有client返回失败，就会释放已经加过的锁，然后进入阻塞的加锁，从第一个节点开始加锁。
nonblock_entrylk同上
AFR_ENTRY_RENAME_TRANSACTION 的锁稍微有点不同：
第一，首先对当前的目录进行阻塞式的加锁，依次加锁。如果当前目录加锁成功，会对其上层目录进行加锁。
 
解锁
无论之前使用阻塞或者非阻塞的加锁，这里都用非阻塞的方式进行解锁。有一点疑惑的是，解锁也可能会出现失败。但是这里并没有对解锁失败进行处理，只是打印了一条日志。
写文件
1、对文件进行加锁
3.  设置xattr的信息
3、在所有up子卷上写文件，op_ret，pre_buf和post_buf设置为第一个返回成功的子卷的值，op_errno返回最后一个子卷的返回值。如果有子卷返回失败，将其pending的值改为0。
5. 重新设置xattr信息
6. 解锁
 
读文件
由于每个子卷对应的内容都一样，当读取文件的时候，从他们任意一个节点读取相应的文件。
如果选取的第一个节点读取失败，会依次选取下一个子卷，直到没有子卷可以选择。
 
open_dir
读文件之前会进行open_dir的操作，open_dir会对所有的child_up的节点做打开操作。
open_dir的call_back中，会对所有的节点做检查（对所有的子目录做检查）。
检查时，首先做一个简单的检查：
遍历目录中所有的文件或目录，将目录中的文件名做一个简单的checksum（两种：1. 根据算法将字符相加，2. 计算MD5）
目录中所有的文件名取^
如果有子卷的checksum值和其他的不一样，则说明有问题
进入self_heal
weak_checksum（Mark Adler's Adler-32 checksum）,注意：这个算法仅仅用来计算pathnames的checksum，他们不需要处理过长string of data
 
创建文件夹
对每个子卷加非阻塞项锁，加锁成功的子卷数等于up子卷，才算加锁成功，否则会解除之前加成功的锁；
如果非阻塞锁失败了，会调用阻塞锁进行加锁；
在所有up子卷上创建文件夹，中间可能有创建失败的文件夹，op_errno返回最后一个子卷写返回的错误信息
设置xattr中的待定标识；
解锁
 
删除文件夹
对每个子卷加非阻塞项锁，加锁成功的子卷数等于up子卷，才算加锁成功，否则会解除之前加成功的锁；
如果非阻塞锁失败了，会调用阻塞锁进行加锁；
对所有up子卷发送删除文件夹的操作，中间可能有删除文件夹失败的操作，op_ret为第一个删除成功返回的op_ret;preparent,postparent为read_child节点返回的参数；op_errno为最后一次rmdir操作的返回值，可能为成功，也可能为失败
设置父目录xattr中的待定标识；
解锁

删除文件
对每个子卷加非阻塞项锁，加锁成功的子卷数等于up子卷，才算加锁成功，否则会解除之前加成功的锁；
如果非阻塞锁失败了，会调用阻塞锁进行加锁；
对所有up子卷发送删除文件夹的操作，中间可能有删除文件夹失败的操作，op_ret为第一个删除成功返回的op_ret;preparent,postparent为read_child节点返回的参数；op_errno为最后一次rmdir操作的返回值，可能为成功，也可能为失败；
设置父目录xattr中的待定标识
解锁
 
从上面的描述可知，changelog和xattr是确保镜像文件系统正常工作的重要保证，这里详细讲述下changelog的更改过程。
以写文件为例：
afr_changelog_pre_op
构造了一个n*3的二维数组，每一行表示一个child，3列分别表示data，meta，entry做出了改变。例如当前是写文件，则将数组的第一列全部置为1
构建一个dict的数组，将之前的二维数组序列化到xattr[i]中
依次对所有加锁成功的client进行GF_XATTROP_ADD_ARRAY操作。
afr_changelog_pre_op_cbk
如果返回成功，则会将local->transaction.pre_op[child_index] = 1；后面的文件的写操作需要判断此标记
如果有brick上不支持xattrop的操作，此处写操作直接终止。
如果有个别brick返回失败，忽略
 
afr_changelog_post_op
post_op和pre_op的操作基本一样，写成功的brick对应的位置填为-1，否则置为0，然后进行GF_XATTROP_ADD_ARRAY的操作。
 
changelog和xattr
 
以写文件为例
pre_op
客户端维护了一个pending_dict的二维数组，数组的行数等于子卷的个数，数组的列数等于3（0-data，1-metadata，2-entry & entry rename）
客户端将pending_dict序列化，并存入xattr的数组
客户端将set_attr的请求发给子卷，子卷的attr加上当前的xattr，注意没加锁的子卷不进行该操作
post_op
写操作进行完以后，会对changelog进行-1的操作。
注意：该处未考虑通信失败的情况
changelog可能发生失败，xattr写失败的机器不会进行写操作。