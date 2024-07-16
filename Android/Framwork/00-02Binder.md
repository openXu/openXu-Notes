https://www.jianshu.com/p/a4469ec6b986



ServiceManager是整个binder机制的守护进程，它是server和cilent之间沟通的桥梁。 ServiceManager，Server，Cilent三者运行在不同的进程, ServiceManager在充当守护进程的同时，它也在充当Server。



## 1、ServiceManager启动

ServiceManager代码目录[frameworks/native/cmds/servicemanager/](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/)

servicemanager进程是在zygote进程中启动的，init进程读取了[servicemanager.rc](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/servicemanager.rc)文件后就会启动servicemanager进程

```c++
//servicemanager是服务名称，/system/bin/servicemanager最终会执行 /frameworks/native/cmds/servicemanager/service_manager.c
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    //onrestart代表的是servicemanager进程重启的化会重启的进程
    onrestart restart healthd
    onrestart restart zygote
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

[servicemanager](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/service_manager.c#382启动后做了下面几件事：

```c++
//[frameworks/native/cmds/servicemanager/service_manager.c]
int main(int argc, char** argv){
   //binder_state结构体保存了
   struct binder_state *bs;
   driver = "/dev/binder";
   //打开设备文件 /dev/binder 以及将它映射到本进程的地址空间
   bs = binder_open(driver, 128*1024);
   //将自己注册为所有服务的大管家.
   binder_become_context_manager(bs))
   //进入loop循环等待和处理 Client 进程的通信请求
   binder_loop(bs, svcmgr_handler);
}

//binder_state结构体位于frameworks/native/cmds/servicemanager/binder.c
struct binder_state{
    int fd;  //fd是文件描述符，即表示打开的/dev/binder设备文件描述符
    void *mapped;   //mapped是把设备文件/dev/binder映射到进程空间的起始地址；
    size_t mapsize; //mapsize是内存映射空间的大小
};
```

### 1.1、[binder_open()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/binder.c?fi=binder_open#binder_open)

调用内核binder驱动的open()函数打开binder设备文件，并记录文件描述符；然后调用mmap()函数将文件映射到进程空间其实地址。

```c++
//frameworks/native/cmds/servicemanager/binder.c
struct binder_state *binder_open(const char* driver, size_t mapsize){
		struct binder_state *bs; 
		struct binder_version vers;
    //binder_state成员有：open /dev/binder的fd，mmap映射的大小，mmap后返回的buffer指针mapped；
		bs = malloc(sizeof(*bs));
    //调用kernel binder driver的binder_open；
		bs->fd = open(driver, O_RDWR | O_CLOEXEC);
		bs->mapsize = mapsize;   //⭐️mmap映射大小128kb
    //调用kernel binder driver的binder_mmap。mmap后返回的buffer指针mapped
		bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
		return bs;
}
```

### 1.2、 [binder_become_context_manager()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/binder.c?fi=binder_become_context_manager#147)

通过ioctl发送BINDER_SET_CONTEXT_MGR的cmd给driver层，通知driver层它是context manager，这样别的进程就能调用到ServiceManager了。

```c++
int binder_become_context_manager(struct binder_state *bs){
    struct flat_binder_object obj;
    memset(&obj, 0, sizeof(obj));
    obj.flags = FLAT_BINDER_FLAG_TXN_SECURITY_CTX;
    int result = ioctl(bs->fd, BINDER_SET_CONTEXT_MGR_EXT, &obj);
    // fallback to original method
    if (result != 0) {
        android_errorWriteLog(0x534e4554, "121035042");

        result = ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
    }
    return result;
}
```

### 1.3、 binder_loop()

开启for循环，不停的给driver层发消息，然后读取driver层返回的命令解析后执行相应工作

```c++
void binder_loop(struct binder_state *bs, binder_handler func){
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];
    //发送BC_ENTER_LOOPER给driver层
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));
   //启动一个for循环，在循环中不断重复：发送消息给driver层，和读取driver层返回的消息，调用binder_parse解析返回的消息。
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
      ...
    }
}
```

### 1.4、[binder_parse()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/binder.c?fi=binder_become_context_manager#229)

```c++
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func){
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;
    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        switch(cmd) {
        case BR_NOOP:
            break;
        case BR_TRANSACTION_COMPLETE:
        case BR_TRANSACTION_SEC_CTX:
        case BR_TRANSACTION: 
        case BR_DEAD_BINDER: 
        case BR_FAILED_REPLY:
        case BR_DEAD_REPLY:
    }
    return r;
}
```



## 2、获取servicemanager服务的代理对象

servicemanager服务进程是运行在单独的进程中的，当Client端需要调用某个服务时，都要首先通过binder跨进程调用servicemanager服务去查询这个服务，然后才能使用该服务。不管是服务启动后注册服务动作，还是Client需要使用服务的查询服务动作，都要和servicemanager服务进行Binder通信。

[ServiceManager.java](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/os/ServiceManager.java#160)工具类中有一系列static方法用于addService注册和getService查询服务，它们都基于[getIServiceManager()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/os/ServiceManager.java#105)方法，该方法用于获取servicemanager服务的代理对象：

```java
 private static IServiceManager getIServiceManager() {
   if (sServiceManager != null) {
     return sServiceManager;
   }
   // ⭐️调用BinderInternal.getContextObject()获取servicemanager服务的Binder对象，然后转换成ServiceManagerNative客户端代理对象，后面注册和查找服务就是通过该对象发送binder消息（命令）给servicemanager服务
   sServiceManager = ServiceManagerNative.asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
   return sServiceManager;
 }
```

[BinderInternal.getContextObject()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/com/android/internal/os/BinderInternal.java?fi=BinderInternal#162)获取IBinder对象，这是一个native方法

```java
public static final native IBinder getContextObject();
```

native方法实现在[frameworks/base/core/jni/android_util_Binder.cpp](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/jni/android_util_Binder.cpp#1033)中：

```c++
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}

// ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/){
   return getStrongProxyForHandle(0);
}
```

最终会调用到[ProcessState.getStrongProxyForHandle()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/libs/binder/ProcessState.cpp#259)：

```c++
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle){
    sp<IBinder> result;
    AutoMutex _l(mLock);
    //查找handle_entry
    handle_entry* e = lookupHandleLocked(handle);
    if (e != nullptr) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                Parcel data;
                //去ping下ServiceManager是否活着，没活着则返回null
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);
                if (status == DEAD_OBJECT)
                   return nullptr;
            }
            // 根据handle来构造BpBinder，这时候的handle是0
            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```



## 3、注册服务

[SystemServer](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/services/java/com/android/server/SystemServer.java)进程启动后，会启动一系列(八九十个)系统服务，这其中就包括AMS、PKMS、WMS等，这些服务被启动之后都会调用[ServiceManager.addService("key", SystemServer)](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/os/ServiceManager.java#160)方法注册服务，方便Client在需要使用这些服务时可通过ServiceManager找到它们。

```java
//通过binder跨进程调用servicemanager服务，注册服务
public static void addService(String name, IBinder service) {
   try {
     getIServiceManager().addService(name, service, false);
   } catch (RemoteException e) {
     Log.e(TAG, "error in addService", e);
   }
 }
```

[ServiceManagerNative.addService()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/base/core/java/android/os/ServiceManagerNative.java#147)

```java
 public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority)
            throws RemoteException {
   Parcel data = Parcel.obtain();
   Parcel reply = Parcel.obtain();
   data.writeInterfaceToken(IServiceManager.descriptor);
   data.writeString(name);
   data.writeStrongBinder(service);
   data.writeInt(allowIsolated ? 1 : 0);
   data.writeInt(dumpPriority);
   mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
   reply.recycle();
   data.recycle();
 }
```

servicemanager服务启动后最后会进入loop循环获取消息，当收到消息后会调用[binder_parse()](http://aospxref.com/android-10.0.0_r47/xref/frameworks/native/cmds/servicemanager/binder.c#229)解析消息：

```c++
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        switch(cmd) {
        case BR_TRANSACTION: {
            struct binder_transaction_data_secctx txn;
            binder_dump_txn(&txn.transaction_data);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, &txn.transaction_data);
                //调用func进行具体消息处理
                res = func(bs, &txn, &msg, &reply);
                //异步则告知driver层释放buffer
                if (txn.transaction_data.flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn.transaction_data.data.ptr.buffer);
                } else {
                    //同步则发送回复消息，回复消息在reply中
                    binder_send_reply(bs, &reply, txn.transaction_data.data.ptr.buffer, res);
                }
            }
            break;
        }
    }
    return r;
}
```

上面代码会调用func(bs,txn,&msg,&reply)方法，func是service_manager.c中的svcmgr_handler方法，来看下这方法（只展示了与注册服务相关代码）

```c++
service_manager.c
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint_t *s;
    size_t len;
    uint_t handle;
    uint_t strict_policy;
    int allow_isolated;


    if (txn->target.ptr != BINDER_SERVICE_MANAGER)
        return -;

    if (txn->code == PING_TRANSACTION)
        return ;

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint(msg);
    s = bio_get_string(msg, &len);
    if (s == NULL) {
        return -;
    }

    if ((len != (sizeof(svcmgr_id) / )) ||
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s\n", str(s, len));
        return -;
    }

    if (sehandle && selinux_status_updated() > ) {
        struct selabel_handle *tmp_sehandle = selinux_android_service_context_handle();
        if (tmp_sehandle) {
            selabel_close(sehandle);
            sehandle = tmp_sehandle;
        }
    }

    switch(txn->code) {
    省略代码...
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string(msg, &len);
        if (s == NULL) {
            return -;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint(msg) ?  : ;
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -;
        break;

    省略代码...
    }

    bio_put_uint(reply, );
    return ;
}
```

注册服务的code值是SVC_MGR_ADD_SERVICE，因此进入这个case中，从msg中解析出s和handle，进而在调用do_add_service方法，看下相关代码

```java
service_manager.c
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;

    if (!handle || (len == 0) || (len > 127))
        return -1;

  //判断是否有权限注册，
    if (!svc_can_register(s, len, spid, uid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }

  //查找svcinfo，它是一个封装了服务信息的结构体
    si = find_svc(s, len);
  //存在，则给si的handle赋新的handle值
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);
        }
        si->handle = handle;
    } else {
    //不存在则构建si
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
    //把si放入svclist链表中
        svclist = si;
    }

    binder_acquire(bs, handle);
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

上面代码主要是做了添加当前的service信息到svclist链表中的过程：若查找到si（封装了服务信息的结构体），则修改它的handle值；否则构造一个si，放入svclist链表中。

svcinfo
来看下svcinfo包含的数据有哪些，下面是它的代码

```c++

struct svcinfo
{   
    //指向下个节点
    struct svcinfo *next;
    //handle值
    uint32_t handle;
    //死亡相关d的结构体
    struct binder_death death;
    int allow_isolated;
    //下面是服务d的名称
    size_t len;
    uint16_t name[0];
};
```

svclist是svcinfo的链表，注册的服务存储在svcinfo中后，最终会以链表的形式存在。

注册服务成功后，会给添加服务的进程发送reply信息，到此整个流程结束，不知大家是否发现ServiceManager中保存的服务，服务的信息主要是name和handle（BpBinder）值，并没有保存服务的真正对象引用。



## 4、获取服务

Context.getSystemService("name")都是由[ContextImpl](http://aospxref.com/android-12.0.0_r3/xref/frameworks/base/core/java/android/app/ContextImpl.java)实现的，每个Application、Activity、Service都有一个ContextImpl，他们是在[ActivityThread.handleBindApplication()](http://aospxref.com/android-12.0.0_r3/xref/frameworks/base/core/java/android/app/ActivityThread.java#6626)、[ActivityThread.performLaunchActivity()](http://aospxref.com/android-12.0.0_r3/xref/frameworks/base/core/java/android/app/ActivityThread.java#3532)中实例化的。

- [ContextImpl.getSystemService("name")](http://aospxref.com/android-12.0.0_r3/xref/frameworks/base/core/java/android/app/ContextImpl.java#2048)

- SystemServiceRegistry.getSystemService(this, name)

所有获取系统服务的Binder对象都要通过ServiceManager.getService()静态方法，比如：

```java
final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
return IActivityTaskManager.Stub.asInterface(b);
```

```kotlin
ServiceManager.java
public static IBinder getService(String name) {
  try {
    IBinder service = sCache.get(name);
    if (service != null) {
      return service;
    } else {
      return getIServiceManager().getService(name);
    }
  } catch (RemoteException e) {
    Log.e(TAG, "error in getService", e);
  }
  return null;
}
```

ServiceManager中会有一个静态sCache来保存所有的服务，不存在则通过binder进程通信去ServiceManager中获取，获取服务的binder请求最终会到service_manager.c中的svcmgr_handler方法，看下相关代码

```c++
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;

    省略代码...

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;

    省略代码...
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

获取服务的code值是SVC_MGR_GET_SERVICE，因此进入这个case，会调用do_find_service方法，来看下这个方法

```cpp
uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    struct svcinfo *si = find_svc(s, len);

    if (!si || !si->handle) {
        return 0;
    }

    if (!si->allow_isolated) {
        // If this service doesn't allow access from isolated processes,
        // then check the uid to see if it is isolated.
        uid_t appid = uid % AID_USER;
        if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
            return 0;
        }
    }

    if (!svc_can_find(s, len, spid, uid)) {
        return 0;
    }

    return si->handle;
}
```



这方法很简单就是从svclist中查找到svcinfo后把svcinfo的handle返回，进而在调用 bio_put_ref(reply, handle) 把handle放入reply中，在给获取服务的进程发送reply信息。获取服务的进程最终获取到的是一个（BinderProxy或BpBinder），BpBinder中的handle在driver层是会被再次转换为当前进程的handle值的。
