


[AndroidXRef](http://androidxref.com/)项目提供 Android 源码的交叉索引，可以快速的搜索符合特定条件的Android源代码，后台是基于 OpenGrok 引擎，OpenGrok是一个快速，便于使用的源码搜索引擎与对照引擎，它能够帮助我们快速的搜索、定位、对照代码树。AndroidXRef提供了完整的Android kernel源码的索引。

# 1. Binder驱动程序


Binder驱动程序实现在内核空间中，它主要由binder.h和binder.c两个源文件组成。下面介绍Binder驱动程序的基础知识，包括基础数据结构、初始化操作、以及设备文件/dev/binder的打开(open)、内存映射(mmap)和内核缓冲区管理等操作。

[Binder内核驱动源码androidxref](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/)

## 1.1 基础数据结构

[binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)

### binder_work

```cpp
struct binder_work {
	struct list_head entry;
	enum {
		BINDER_WORK_TRANSACTION = 1,
		BINDER_WORK_TRANSACTION_COMPLETE,
		BINDER_WORK_NODE,
		BINDER_WORK_DEAD_BINDER,     //标志一个具体的死亡通知类型
		BINDER_WORK_DEAD_BINDER_AND_CLEAR,//标志一个具体的死亡通知类型
		BINDER_WORK_CLEAR_DEATH_NOTIFICATION,//标志一个具体的死亡通知类型
	} type;
};
```

binder_work结构体用来描述待处理的工作项，成员变量entry用于将该结构体嵌入到一个宿主结构中，type用于描述工作项的类型，根据type的取值，Binder驱动程序就可以判断出一个binder_work到底嵌入到什么类型的宿主结构中。

### binder_node

```cpp
struct binder_node {
	int debug_id;
	struct binder_work work;
	union {
		struct rb_node rb_node;
		struct hlist_node dead_node;
	};
	//★指向一个Binder实体对象的宿主进程，Binder驱动中宿主进程通过binder_proc描述
	struct binder_proc *proc;    
	//★引用了该Binder实体的所有Client引用保存在hash列表中
	struct hlist_head refs;      
	int internal_strong_refs;    //描述一个Binder实体对象的强引用计数
	int local_weak_refs;         //描述一个Binder实体对象的弱引用计数
	int local_strong_refs;       //描述一个Binder实体对象的强引用计数 
	binder_uintptr_t ptr;        //指向Service组件内部的一个引用计数对象(weakref_impl)的地址
	binder_uintptr_t cookie;     //指向Service组件的地址
	unsigned has_strong_ref:1;
	unsigned pending_strong_ref:1;
	unsigned has_weak_ref:1;
	unsigned pending_weak_ref:1;
	unsigned has_async_transaction:1;  //描述一个Binder实体对象是否正在处理一个异步事务，1表示是，0表示否
	unsigned accept_fds:1;   //描述一个Binder实体对象是否可以接受包含有文件描述符的进程间通信数据
	unsigned min_priority:8;    //表示Binder实体对象在处理一个来自Client进程请求时，它所要求的处理线程，即Service进程中的一个线程
	struct list_head async_todo;
};
```

binder_node描述一个Binder实体对象，每个Service组件在Binder驱动程序中都对应一个Binder实体对象，用于描述它在内核中的状态，Binder驱动程序通过强引用计数和弱引用计数技术来维护他们的声明周期。

### binder_ref_death

```cpp
struct binder_ref_death {
	//标志一个具体的死亡通知类型，取值请看binder_work结构体中的枚举值
	struct binder_work work;   
	binder_uintptr_t cookie;   //保存负责接收死亡通知的对象地址
};
```

描述一个Service组件的死亡接收通知。如果Service组件所在进程意外崩溃，Client进程能获得它所引用的Service组件死亡通知，以便做出相应处理，Client进程需要将一个用于接收通知的对象地址注册到Binder驱动程序中。

### binder_ref

```cpp
struct binder_ref {
	/* Lookups needed: */
	/*   node + proc => ref (transaction) */
	/*   desc + proc => ref (transaction, inc/dec ref) */
	/*   node => refs + procs (proc exit) */
	int debug_id;
	struct rb_node rb_node_desc;
	struct rb_node rb_node_node;
	struct hlist_node node_entry;
	//proc指向Binder引用对象的宿主进程
	struct binder_proc *proc;
	//node描述binder引用对象所引用的binder实体对象，每个引用都将保存在Binder实体对象的hash列表中 (binder_node结构体的成员变量refs )，binder引用对象的成员变量node_entry正好是hash列表的节点
	struct binder_node *node;  
	uint32_t desc; //描述一个Binder引用对象的句柄值，当Client进程通过Binder驱动访问一个Service组件，只需要指定一个句柄值，Binder驱动根据句柄值找到对应Binder引用对象，再根据引用对象的node找到对应的Binder实体对象，最后通过实体对象找到访问的Service组件
	int strong;
	int weak;
	//death指向一个Service组件的死亡接收通知
	struct binder_ref_death *death;
};
```

binder_ref描述一个Binder引用对象，每个Client组件在Binder驱动程序中都应用有一个Binder引用对象，用来表述它在内核中的状态。

### binder_buffer

```cpp
struct binder_buffer {
	//entry是内核缓冲区列表的一个节点
	struct list_head entry; /* free and allocated entries by address */
	struct rb_node rb_node; /* free entry by size or allocated entry */
				/* by address */
	unsigned free:1;   //该内核缓冲区是否空闲
	//如果Service组件处理完该事务，allow_user_free==1时会请求Binder驱动释放该内核缓冲区
	unsigned allow_user_free:1;
	//1：内核缓冲区关联的是一个异步事务
	unsigned async_transaction:1;  
	unsigned debug_id:29;  //标志内核缓冲区的身份
	//描述内核缓冲区正交给哪个事务处理。每个事务都关联一个目标Binder实体对象，Binder驱动程序将事务数据保存在内核缓冲区中，然后将它交给目标Binder实体对象处理，目标Binder实体对象再将该内核缓冲区的内容交给相应Service组件处理
	struct binder_transaction *transaction;  
	//描述内核缓冲区正交给哪个Binder实体对象使用 
	struct binder_node *target_node;   
	size_t data_size;   //数据缓冲区大小
	size_t offsets_size; //偏移数组大小，偏移数组记录每个Binder对象在数据缓冲区中的位置
	//指向一块大小可变的数据缓冲区，真正用来保存通信数据的
	uint8_t data[0];    
};
```

binder_buffer表述一个内核缓冲区，它是用来在进程间传输数据的。每个使用Binder IPC的进程在Binder驱动中都有一个内核缓冲区列表，用来保存Binder驱动程序为它所分配的内核缓冲区。

### binder_proc

```cpp
struct binder_proc {
	struct hlist_node proc_node;  //hash列表节点
	struct rb_root threads;       //binder线程池红黑树的根节点
	struct rb_root nodes;
	struct rb_root refs_by_desc;
	struct rb_root refs_by_node;
	int pid;   //进程的进程组ID
	struct vm_area_struct *vma;  //用户空间地址
	struct mm_struct *vma_vm_mm;
	struct task_struct *tsk;    //任务控制块
	struct files_struct *files; //打开文件结构体数组
	struct hlist_node deferred_work_node;
	int deferred_work;
	void *buffer;       //内核空间地址
	ptrdiff_t user_buffer_offset;

	struct list_head buffers;
	struct rb_root free_buffers;
	struct rb_root allocated_buffers;
	size_t free_async_space;
	//物理页面
	struct page **pages;
	//内核缓冲区的大小
	size_t buffer_size;
	uint32_t buffer_free;
	struct list_head todo;
	wait_queue_head_t wait;
	struct binder_stats stats;
	struct list_head delivered_death;
	int max_threads;     //Binder驱动最多可主动请求进程注册的线程数量
	int requested_threads;
	int requested_threads_started;
	int ready_threads;   //进程当前的空闲Binder线程数目
	long default_priority;
	struct dentry *debugfs_entry;
};
```

binder_proc用来描述一个正在使用Binder进程间通信机制的进程。当一个进程调用open打开设备文件/dev/binder时，Binder驱动就会为它创建一个binder_proc结构体，并将它保存在一个全局的hash列表中，成员变量proc_node正好是hash列表中的一个节点。

进程打开设备文件后必须调用mmap函数将它映射到进程的地址空间，实际上是请求Binder驱动为进程分配内核缓冲区，buffer_size保存内核缓冲区的大小。内核缓冲区有两个地址，一个是内核空间地址，另一个为用户空间地址，内核空间地址保存在buffer中，用户空间地址保存在vma中，这两个地址相差一个固定的值，保存在成员变量user_buffer_offset中。这样，给定一个用户空间地址或者内核空间地址，Binder驱动就可以计算出另一个地址的大小。

两个空间地址都是虚拟地址，他们对应的物理页面保存在成员变量pages中。

### binder_thread

```cpp
struct binder_thread {
	struct binder_proc *proc;   //指向宿主进程
	struct rb_node rb_node;    //binder_proc进程使用红黑树组织Binder线程池中的线程，rb_node就是该红黑树的一个节点
	int pid;   //线程id
	int looper;//线程状态
	struct binder_transaction *transaction_stack;
	struct list_head todo;
	uint32_t return_error; /* Write failed, return error code in read buf */
	uint32_t return_error2; /* Write failed, return error code in read */
		/* buffer. Used when sending a reply to a dead process that */
		/* we are also waiting on */
	wait_queue_head_t wait;
	struct binder_stats stats;
};
```

binder_thread描述Binder线程池中的一个线程

### binder_transaction

```cpp
struct binder_transaction {
	int debug_id;
	struct binder_work work;        
	struct binder_thread *from;     //事务发起的线程（源线程）
	struct binder_transaction *from_parent;   //事务依赖
	struct binder_proc *to_proc;    //负责处理该事务的进程（目标进程）
	struct binder_thread *to_thread;//负责处理该事务的线程（目标线程）
	struct binder_transaction *to_parent;//事务依赖
	unsigned need_reply:1;    //区分事务是同步1还是异步0的
	/* unsigned is_dead:1; */	/* not used at the moment */
	//指向Binder驱动为该事务分配的一块内核缓冲区，它里面保存进程间通信数据
	struct binder_buffer *buffer;  
	unsigned int	code;  //直接从通信数据中拷贝过来的
	unsigned int	flags; //直接从通信数据中拷贝过来的
	long	priority;     //源线程的优先级
	long	saved_priority; //Binder驱动修改线程优先级时用于保存原来优先级的
	kuid_t	sender_euid;  //源线程用户ID
};
```

binder_transaction描述进程间通信过程，这个过程又称为一个事务

以上结构体都是在Binder驱动程序内部使用的。应用程序进程在打开设备文件/dev/binder后需要通过IO控制函数ioctl来进一步与Binder驱动程序交互，Binder驱动程序提供了一系列IO控制命令来和应用程序通信，在这些IO控制命令中，最重要的就是`BINDER_WRITE_READ`命令了，它的定义如下：



### BINDER_WRITE_READ & binder_write_read

[binder.h](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/uapi/binder.h)

```cpp
//IO控制命令BINDER_WRITE_READ后跟的参数是一个binder_write_read结构体
#define BINDER_WRITE_READ		_IOWR('b', 1, struct binder_write_read)

struct binder_write_read {
	//描述输入数据（用户控件传到Binder驱动程序的数据）
	binder_size_t		write_size;	/* 缓冲区大小，单位字节 */
	binder_size_t		write_consumed;	/*binder驱动从缓冲区处理了多少个字节*/
	binder_uintptr_t	write_buffer;  //指向一个用户空间缓冲区地址
	//描述输出数据（从Binder驱动返回给用户空间的数据）
	binder_size_t		read_size;	/* bytes to read */
	binder_size_t		read_consumed;	/* bytes consumed by driver */
	binder_uintptr_t	read_buffer; //指向一个用户空间缓冲区地址
};
```

binder_write_read结构体用来描述进程间通信过程中传输的数据，包括输入数据和输出数据。缓冲区数据是有一定的数据格式的，它们由命令协议码和返回协议码来控制的。


###  xxx

```cpp

```

## 1.2 Binder设备初始化过程

Binder设备的初始化过程是在Binder驱动程序的初始化函数binder_init中进行的。

[binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)

```cpp
static int __init binder_init(void)
{
	int ret;
	//在目标设备上创建一个/proc/binder/proc目录，每个使用Binder IPC的进程在该目录下都对应一个文件，这些文件以进程ID命名，可以通过它们读取各进程Binder线程池、Binder实体对象、Binder引用对象以及内核缓冲区等信息
	binder_deferred_workqueue = create_singlethread_workqueue("binder");
	if (!binder_deferred_workqueue)
		return -ENOMEM;

	binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
	if (binder_debugfs_dir_entry_root)
		binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
						 binder_debugfs_dir_entry_root);
	//创建一个Binder设备(misc_register函数创建一个misc类型的字符设备)，全局变量binder_miscdev的定义在下面
	ret = misc_register(&binder_miscdev);
	//在/proc/binder目录下创建5个文件，通过这5个文件可以读取到Binder驱动程序的运行状况
	if (binder_debugfs_dir_entry_root) {
		debugfs_create_file("state",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_state_fops);
		debugfs_create_file("stats",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_stats_fops);
		debugfs_create_file("transactions",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    NULL,
				    &binder_transactions_fops);
		debugfs_create_file("transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    &binder_transaction_log,
				    &binder_transaction_log_fops);
		debugfs_create_file("failed_transaction_log",
				    S_IRUGO,
				    binder_debugfs_dir_entry_root,
				    &binder_transaction_log_failed,
				    &binder_transaction_log_fops);
	}
	return ret;
}
//全局变量binder_fops指定设备文件的操作方法列表
static const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl, //IO控制函数
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,   //内存映射方法
	.open = binder_open,   //打开设备文件/dev/binder
	.flush = binder_flush,
	.release = binder_release,
};
static struct miscdevice binder_miscdev = {
	.minor = MISC_DYNAMIC_MINOR,
	.name = "binder",
	.fops = &binder_fops  
};

```

## 1.3 Binder设备文件打开过程

一个进程在使用Binder IPC之前，首先要调用函数open打开设备文件/dev/bidner来获得一个文件描述符，然后才能通过这个文件描述符来和Binder启动程序交互。进程调用open函数时Binder驱动程序中的binder_open函数就会被调用：

```cpp
static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
		     current->group_leader->pid, current->pid);
	//为进程创建一个binder_proc结构体，并对它进行初始化
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	if (proc == NULL)
		return -ENOMEM;
	get_task_struct(current);
	proc->tsk = current;
	INIT_LIST_HEAD(&proc->todo);
	init_waitqueue_head(&proc->wait);
	proc->default_priority = task_nice(current);

	binder_lock(__func__);

	binder_stats_created(BINDER_STAT_PROC);
	//将binder_proc结构体对象加入到一个全局的hash队列binder_procs中
	hlist_add_head(&proc->proc_node, &binder_procs);
	proc->pid = current->group_leader->pid;
	INIT_LIST_HEAD(&proc->delivered_death);
	filp->private_data = proc;

	binder_unlock(__func__);

	if (binder_debugfs_dir_entry_proc) {
		char strbuf[11];

		snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
		proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
			binder_debugfs_dir_entry_proc, proc, &binder_proc_fops);
	}

	return 0;
}
```

打开设备文件过程中，为进程创建一个binder_proc结构体，并对它进行初始化，然后将其加入到一个全局的hash队列binder_procs中。

## 1.3 Binder设备文件打开过程
