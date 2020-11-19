
https://blog.csdn.net/yiranfeng/article/details/105210069

http://gityuan.com/2015/11/07/binder-start-sm/


# 1. ServiceManager启动过程

[init.rc](https://www.androidos.net.cn/android/10.0.0_r6/xref/system/core/rootdir/init.rc)
```c
# Start essential services.
start servicemanager
```

[frameworks/native/cmds/servicemanager/servicemanager.rc](http://androidos.net.cn/android/10.0.0_r6/xref/frameworks/native/cmds/servicemanager/servicemanager.rc)

```c
service servicemanager /system/bin/servicemanager
    class core animation
    user system    //以系统用户system身份运行
    group system readproc
    critical    //表明servicemanager是系统中的一个关键服务，关键服务是不可以退出的，一旦退出就会被系统重启，如果关键服务在4分钟内退出次数大于4次，系统就会重启
    onrestart restart healthd
    onrestart restart zygote    //如果servicemanager发生重启，需要将zygote进程重新启动
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    onrestart restart thermalservice
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```

ServiceManager是由init进程通过解析init.rc文件而启动的，它对应的进程名为servicemanager，可执行程序为`/system/bin/servicemanager`，这个程序是由`service_manager.c`编译的可执行程序。servicemanager服务启动后会执行`service_manager.c`的mian方法：

[frameworks/native/cmds/servicemanager/service_manager.c](http://androidos.net.cn/android/10.0.0_r6/xref/frameworks/native/cmds/servicemanager/service_manager.c)

```c
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }
    //1. 调用函数binder_open打开设备文件/dev/binder，将它映射到本进程的地址空间
    bs = binder_open(driver, 128*1024);
    ...
    //2. 将自己注册为Binder进程间通信机制的上下文管理者
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }
    ...
    //3. 循环等待处理Client进程的请求(注册binder、获取binder)
    binder_loop(bs, svcmgr_handler);

    return 0;
}
```

## 1.1 打开和映射Binder设备文件

binder_open()函数用于打开设备文件/dev/binder，并将它映射到进程的地址空间，该函数实现如下：

[frameworks/native/cmds/servicemanager/binder.c](http://androidos.net.cn/android/10.0.0_r6/xref/frameworks/native/cmds/servicemanager/binder.c)

```cpp
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;
    ...
    //open函数打开设备文件/dev/binder，这会使得Binder驱动程序中的binder_open函数被调用
    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
    ...
    bs->mapsize = mapsize;
    //调用mmap函数将设备文件/dev/binder映射到进程地址空间，请求映射的地址空间大小为128k
    //binder驱动程序会为进程分配128k大小的内核缓冲区
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }
    //binder_state记录了映射后的地址空间起始地址和大小，将其返回
    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```

## 1.2 注册为Binder上下文管理者

ServiceManager通过IO控制命令`BINDER_SET_CONTEXT_MGR`将自己注册到Binder驱动程序中，由函数`binder_become_context_manager()`实现

[frameworks/native/cmds/servicemanager/binder.c](http://androidos.net.cn/android/10.0.0_r6/xref/frameworks/native/cmds/servicemanager/binder.c)

```cpp
int binder_become_context_manager(struct binder_state *bs)
{
    struct flat_binder_object obj;
    memset(&obj, 0, sizeof(obj));
    obj.flags = FLAT_BINDER_FLAG_TXN_SECURITY_CTX;

    int result = ioctl(bs->fd, BINDER_SET_CONTEXT_MGR_EXT, &obj);

    // fallback to original method
    if (result != 0) {
        android_errorWriteLog(0x534e4554, "121035042");
        //注册为Binder上下文管理者
        result = ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
    }
    return result;
}
```

IO控制命令`BINDER_SET_CONTEXT_MGR`的定义在[kernel_3.18/xref/drivers/staging/android/uapi/binder.h](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/uapi/binder.h)

```cpp
#define BINDER_SET_CONTEXT_MGR		_IOW('b', 7, __s32)
```

Binder驱动程序是在它的binder_ioctl中处理IO控制命令`BINDER_SET_CONTEXT_MGR`的

[kernel_3.18/xref/drivers/staging/android/binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)

```cpp
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	thread = binder_get_thread(proc);
	switch (cmd) {
		...
    	case BINDER_SET_CONTEXT_MGR:
    	//★ 继续调用binder_ioctl_set_ctx_mgr()
		ret = binder_ioctl_set_ctx_mgr(filp);
		break;
		...
	}
	...
	return ret;
}

//service manager所对应的binder_node
static struct binder_node *binder_context_mgr_node;
//运行service manager的线程uid
static kuid_t binder_context_mgr_uid = INVALID_UID;

static int binder_ioctl_set_ctx_mgr(struct file *filp)
{
	int ret = 0;
	//为ServiceManager进程创建一个binder_proc结构体
	struct binder_proc *proc = filp->private_data;
	kuid_t curr_euid = current_euid();

	if (binder_context_mgr_node != NULL) {
		pr_err("BINDER_SET_CONTEXT_MGR already set\n");
		ret = -EBUSY;
		goto out;
	}
	if (uid_valid(binder_context_mgr_uid)) {
		...
	} else {
		binder_context_mgr_uid = curr_euid;
	}
	//★ 创建ServiceManager的binder实体
	binder_context_mgr_node = binder_new_node(proc, 0, 0);
	if (binder_context_mgr_node == NULL) {
		ret = -ENOMEM;
		goto out;
	}
	binder_context_mgr_node->local_weak_refs++;
	binder_context_mgr_node->local_strong_refs++;
	binder_context_mgr_node->has_strong_ref = 1;
	binder_context_mgr_node->has_weak_ref = 1;
out:
	return ret;
}

//创建binder实体对象
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       binder_uintptr_t ptr,
                       binder_uintptr_t cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;

    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node);

        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }
    //给新创建的binder_node分配内核空间
    node = kzalloc(sizeof(*node), GFP_KERNEL);
    if (node == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_NODE);
    // 将新创建的node对象添加到proc红黑树；
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
    node->work.type = BINDER_WORK_NODE;
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
             "%d:%d node %d u%016llx c%016llx created\n",
             proc->pid, current->pid, node->debug_id,
             (u64)node->ptr, (u64)node->cookie);
    return node;
}
```

## 1.3 循环等待Client进程请求

ServiceManager需要在系统运行期间为Service组件和Client组件提供服务，所以通过一个无线循环来等待和处理Service组件和Client组件的进程间通信请求，这是通过`binder_loop`实现的

[frameworks/native/cmds/servicemanager/binder.c](http://androidos.net.cn/android/10.0.0_r6/xref/frameworks/native/cmds/servicemanager/binder.c)

```cpp
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        //不断使用IO控制命令BINDER_WRITE_READ来检查Binder驱动程序是否有新的进程间通信请求需要它来处理
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```

# ServiceManager代理对象的获取过程

https://blog.csdn.net/Luoshengyang/article/details/6627260

ServiceManager在Binder机制中用于管理所有的Server

Server和Client是如何获得Service Manager接口的？即defaultServiceManager 接口是如何实现的





# Service组件的启动过程

3. Server 是如何把自己的服务启动起来的？Service Manager 在 Server
启动的过程中是如何为 Server 提供服务的？即 IServiceManager::addService
接口是如何实现的。



# Service代理对象的获取过程

 Service Manager 是如何为 Client 提供服务的？即
IServiceManager::getService 接口是如何实现的


# 总结


## ServiceManager启动过程

ServiceManger集中管理系统内的所有服务，通过权限控制进程是否有权注册服务,通过字符串名称来查找对应的Service。servicemanager进程是由init进程创建的，servicemanager进程启动后做了3件事

- 调用函数binder_open打开设备文件/dev/binder，调用mmap()方法分配128k的内存映射空间映射到本进程的地址空间
- 通过IO控制命令`BINDER_SET_CONTEXT_MGR`将自己注册到Binder驱动程序中，Binder驱动会为它创建binder实体对象，将它注册为Binder进程间通信机制的上下文管理者
- 循环等待处理Client进程的请求(注册binder、获取binder)




















