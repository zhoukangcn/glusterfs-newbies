官方修复的一个BUG分析
====================
参见此url：
https://bugzilla.redhat.com/show_bug.cgi?id=765194
http://review.gluster.com/#/c/3226/
发生的情况如下：
在所有的brick上，pre_op完成，所有的xattr中的changelog被标记为1
文件操作完成后，向所有的clint发出post_op操作
在subvol_0上，client_0上的changelog被改为0，在client_1的changelog被改为1之前，brick挂掉了
所以，对于subvol_0来说，它认为自己成功了，但是subvol_1失败了，此时subvol-1上的changelog写成功
如果有继续的文件写入，在subvol-1上，自己的changelog为0，subvol-0的changelog将不为0
如果subvol-0重新启动的时候，change-log所指示的状态是脑裂状态，但是其实不是。
 
给出的解决方案是：
pre_op应该先写本机器的change_log
post_op应该最后写本机器的change_log
 
上面的意思是，尽量的让changelog去指责自己，而不是指责别人。
 
 
optimistic_change_log
如果写成功的话，changelog会再进行+1，写失败的话，不变。
 
当本机写成功的时候txn_changelog应该最后写，当本机写失败的时候，txn_changelog应该最先写