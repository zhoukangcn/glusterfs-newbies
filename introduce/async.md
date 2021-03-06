异步及回调机制
==============
异步
GlusterFS在Translator的操作中，使用了大量的异步机制。由于对文件的很多细小的操作都会通过网络传给远端服务器完成，这就要求必须使用一种异步的方式来完成这些操作。
GlusterFS在xlator_t里面定义了大量的函数指针，必须说明的是，这些某些函数指针是需要同步的执行的。
之前介绍了，GlusterFS采用一种树形的Translator来完成处理，树形结构的每一层都有自己的数据需要保存。所以GlusterFS实现了一套回调机制。
STACK_WIND和STACK_UNWIND
Translator API的回调机制主要采用STACK_WIND和STACK_UNWIND。这是GlusterFS实现的一种专用的frame stack，用于表示对translators的调用。当你的请求从Translator树的上一层走到下一层的时候，需要将当前的frame保存在frame stack中，然后通过STACK_WIND传给下一个Translator。
当下一层的Translator处理完数据的时候，会调用STACK_WIND时指定的回调函数，这个时候回调函数会使用STACK_UNWIND将frame pop出来，并进行相关的处理。
STACK_WIND_COOKIE
我们注意到，一个客户端可能对应若干个卷，客户端与每个卷的操作都通过一个client xlator_t来完成，这就需要在调用的时候保存一下当前交互的客户端信息。