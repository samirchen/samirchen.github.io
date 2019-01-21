---
layout: post
title: 获取 iOS 应用的内存使用上限
description: 介绍获取 iOS 应用内存使用上限的方法。
category: blog
tag: iOS, memory usage, 内存占用
---

 
<!-- 

关于 Jetsam，我们可以从手机 `设置->隐私->分析`
看到一些系统的日志，会发现手机上有许多 JetsamEvent
开头的日志。打开这些日志，一般会显示一些内存大小，CPU 时间之类数据。

之所以会发生这些 JetsamEvent，主要还是由于 iOS 设备不存在交换分区导致的内存受限，所以 iOS 内核不得不把一些优先级不高或者占用内存过大的应用给杀掉。这些 JetsamEvent 就是系统在杀掉 App 后记录的一些数据信息。

从某种程度来说，JetsamEvent 是一种另类的 Crash 事件，但是在常规的 Crash 捕获工具中，由于iOS上不能捕获的信号量的限制，所以因为内存导致 App 被杀掉是无法被捕获的。

关于 Jetsam 的研究，可以参考 [iOS 内存 abort(Jetsam) 原理探究][3]。

基于这样的背景，那如果我们能获取到当前 App 使用的内存，还能知道 App OOM 导致 crash 的内存上限，我们就可以做一些事情来防止 App 因为 OOM 而 crash。

通过分析 XNU 的代码，发现了一系列跟内存、进程优先级相关的宏命令，如下所示：

```
#define MEMORYSTATUS_CMD_GET_PRIORITY_LIST            1
#define MEMORYSTATUS_CMD_SET_PRIORITY_PROPERTIES      2
#define MEMORYSTATUS_CMD_GET_JETSAM_SNAPSHOT          3
#define MEMORYSTATUS_CMD_GET_PRESSURE_STATUS          4
#define MEMORYSTATUS_CMD_SET_JETSAM_HIGH_WATER_MARK   5    /* Set active memory limit = inactive memory limit, both non-fatal   */
#define MEMORYSTATUS_CMD_SET_JETSAM_TASK_LIMIT        6    /* Set active memory limit = inactive memory limit, both fatal   */
#define MEMORYSTATUS_CMD_SET_MEMLIMIT_PROPERTIES      7    /* Set memory limits plus attributes independently           */
#define MEMORYSTATUS_CMD_GET_MEMLIMIT_PROPERTIES      8    /* Get memory limits plus attributes                 */
#define MEMORYSTATUS_CMD_PRIVILEGED_LISTENER_ENABLE   9    /* Set the task's status as a privileged listener w.r.t memory notifications  */
#define MEMORYSTATUS_CMD_PRIVILEGED_LISTENER_DISABLE  10   /* Reset the task's status as a privileged listener w.r.t memory notifications  */
#define MEMORYSTATUS_CMD_AGGRESSIVE_JETSAM_LENIENT_MODE_ENABLE  11   /* Enable the 'lenient' mode for aggressive jetsam. See comments in kern_memorystatus.c near the top. */
#define MEMORYSTATUS_CMD_AGGRESSIVE_JETSAM_LENIENT_MODE_DISABLE 12   /* Disable the 'lenient' mode for aggressive jetsam. */
#define MEMORYSTATUS_CMD_GET_MEMLIMIT_EXCESS          13   /* Compute how much a process's phys_footprint exceeds inactive memory limit */
#define MEMORYSTATUS_CMD_ELEVATED_INACTIVEJETSAMPRIORITY_ENABLE     14
#define MEMORYSTATUS_CMD_ELEVATED_INACTIVEJETSAMPRIORITY_DISABLE    15
```

这些命令都会以各种各样的形式通过 `memorystatus_control` 进行调用。我们可以写一些测试代码：

```
#import <dlfcn.h>

void *system = dlopen("/usr/lib/system/libsystem_kernel.dylib", RTLD_LAZY);
int (*my_memorystatus_control)(uint32_t command, int32_t pid, uint32_t flags, void *buffer, size_t buffersize) = dlsym(system, "memorystatus_control");
NSLog(@"memorystatus_control: %p", my_memorystatus_control);
int ret = my_memorystatus_control(6, (int32_t) getpid(), 50, NULL, 0);
if (ret == -1) {
    printf("error: %s\n", strerror(errno));
}
printf("ret: %d\n", ret);
```

得到的输出是：

```
memorystatus_control: 0x114322d20
error: Operation not permitted
ret: -1
```

报错了，无权限。

查看源代码，可以看看 [memorystatus_control][4] 的实现：


```
int
memorystatus_control(struct proc *p __unused, struct memorystatus_control_args *args, int *ret) {
    int error = EINVAL;
    os_reason_t jetsam_reason = OS_REASON_NULL;

#if !CONFIG_JETSAM
    #pragma unused(ret)
    #pragma unused(jetsam_reason)
#endif

    /* Need to be root or have entitlement */
    if (!kauth_cred_issuser(kauth_cred_get()) && !IOTaskHasEntitlement(current_task(), MEMORYSTATUS_ENTITLEMENT)) {
        error = EPERM;
        goto out;
    }

    /*
     * Sanity check.
     * Do not enforce it for snapshots.
     */
    if (args->command != MEMORYSTATUS_CMD_GET_JETSAM_SNAPSHOT) {
        if (args->buffersize > MEMORYSTATUS_BUFFERSIZE_MAX) {
            error = EINVAL;
            goto out;
        }
    }

    switch (args->command) {
    case MEMORYSTATUS_CMD_GET_PRIORITY_LIST:
        error = memorystatus_cmd_get_priority_list(args->pid, args->buffer, args->buffersize, ret);
        break;
    case MEMORYSTATUS_CMD_SET_PRIORITY_PROPERTIES:
        error = memorystatus_cmd_set_priority_properties(args->pid, args->buffer, args->buffersize, ret);
        break;
    case MEMORYSTATUS_CMD_SET_MEMLIMIT_PROPERTIES:
        error = memorystatus_cmd_set_memlimit_properties(args->pid, args->buffer, args->buffersize, ret);
        break;

    // ...

    default:
        break;
    }

out:
    return error;
}
```

关键代码是：

```
/* Need to be root or have entitlement */
if (!kauth_cred_issuser(kauth_cred_get()) && !IOTaskHasEntitlement(current_task(), MEMORYSTATUS_ENTITLEMENT)) {
    error = EPERM;
    goto out;
}
```

只有 root 权限或者 entitlements 里面带有 com.apple.private.memorystatus 才能继续执行，否则就会报错。

`IOTaskHasEntitlement` 的实现如下所示：

```
extern "C" boolean_t 
IOTaskHasEntitlement(task_t task, const char * entitlement)
{
    OSObject * obj;
    obj = IOUserClient::copyClientEntitlement(task, entitlement);
    if (!obj) return (false);
    obj->release();
    return (obj != kOSBooleanFalse);
}
```

如果你的手机上跑的系统版本对应的 XNU 不低于 [3789][4] 版本，可以尝试在 entitlements.plist 里补上了 com.apple.private.memorystatus，属性值设置为 YES，然后利用 codesign 进行了重签来绕过检查。对应的 XNU 不高于 [3248][5] 版本，则需要 root。


当我们知道了这些内核调用，其实我们就可以精准的知道一个 App 使用内存的上限。

从上面的列到的 `MEMORYSTATUS_CMD_GET_PRIORITY_LIST` 命令，我们可以获取到进程的优先级列表，以及每个进程的内存水位上线。


`memorystatus_control` 这个内核调用返回的结构体类型如下：

```
typedef struct memorystatus_priority_entry {
    pid_t pid;
    int32_t priority;
    uint64_t user_data;
    int32_t limit;  /* MB */
    uint32_t state;
} memorystatus_priority_entry_t;
```

其中字段 priority 代表进程在 Jetsam 机制中的优先级，limit 代表内存水位上限。


这个结构体是不对外暴露的，不能直接用，不过结构体无非就是一块内存区域，以某些规则进行布局而已。我们可以自制一个对应的结构体来访问内存区域就好：


```
typedef struct my_memorystatus_priority_entry {
        pid_t pid;
        int32_t priority;
        uint64_t user_data;
        int32_t limit;
        uint32_t state;
} my_memorystatus_priority_entry_t;
```

然后，在我们的 App 中增加下面的测试代码：

```
#import <dlfcn.h>

void *system = dlopen("/usr/lib/system/libsystem_kernel.dylib", RTLD_LAZY);
int (*my_memorystatus_control)(uint32_t command, int32_t pid, uint32_t flags, void *buffer, size_t buffersize) = dlsym(system, "memorystatus_control");
int ret = my_memorystatus_control(1, (int32_t) getpid(), 0, buffer, 65534);
if (ret != 0) {
    int count = ret / sizeof(my_memorystatus_priority_entry_t);
    printf("total processes: %d\n", count);
    for (int i = 0; i < count; i++) {
        my_memorystatus_priority_entry_t entry = buffer[i];
        printf("PID: %d, ", entry.pid);
        printf("Priority: %d, ", entry.priority);
        printf("Limit: %d\n", entry.limit);
    }
}
```

然后我们在 Root 权限下运行一下 App（Test）看看：

```
ps aux | grep Test
mobile    3433   0.0  0.5   581912   4824   ??  Ss    1:17PM   0:00.09 /var/mobile/Containers/Bundle/Application/.../Test.app/Test

```


获取到它的 PID 为 3433，再运行一下我们的列表 dump 就可以找到内存的上限了。通过这样的方式，我们获取到的数据如下：


| 机型 | 内存 OOM 上限(MB) | 内存报警上限(MB) | 真实物理内存(MB) |
| ------ | ------ | ------ | ------ |
| iPhone5 | 650 | 600 | 1024 |
| iPhone5s | 650 | 600 | 1024 |
| iPhone6 | 650 | 600 | 1024 |
| iPhone6+ | 650 | 600 | 1024 |
| iPhone6s | 1380 | 1280 | 2048 |
| iPhone6s+ | 1380 | 1280 | 2048 |
| iPhone7 | 1380 | 1280 | 2048 |
| iPhone7+ | 2050 | 1950 | 3072 |
| iPhone8 | 1380 | 1280 | 2048 |
| iPhone8+ | 2050 | 1950 | 3072 |
| iPhone8+ | 2050 | 1950 | 2785 |


内存报警上限的获取代码大致如下：

```
	/*
     * Configure the per-task memory limit warning level.
     * This is computed as a percentage.
     */
    max_task_footprint_warning_level = 0;

  if (max_mem < 0x40000000) {
  /*
   * On devices with < 1GB of memory:
   *    -- set warnings to 50MB below the per-task limit.
   */
  if (max_task_footprint_mb > 50) {
    max_task_footprint_warning_level = ((max_task_footprint_mb - 50) * 100) / max_task_footprint_mb;
  }
} else {
  /*
   * On devices with >= 1GB of memory:
   *    -- set warnings to 100MB below the per-task limit.
   */
  if (max_task_footprint_mb > 100) {
    max_task_footprint_warning_level = ((max_task_footprint_mb - 100) * 100) / max_task_footprint_mb;
  }
}
```


参考：

- [Handling low memory conditions in iOS and Mavericks][6]
- 《深入解析 Mac OS X & iOS 操作系统》
- [iOS memory allocation][7]

-->


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-app-memory-limit
[3]: https://www.aliyun.com/jiaocheng/349522.html
[4]: https://opensource.apple.com/source/xnu/xnu-3789.70.16/bsd/kern/kern_memorystatus.c.auto.html
[5]: https://opensource.apple.com/source/xnu/xnu-3248.40.184/bsd/kern/kern_memorystatus.c.auto.html
[6]: http://www.newosxbook.com/articles/MemoryPressure.html
[7]: https://stackoverflow.com/questions/6044147/ios-memory-allocation-how-much-memory-can-be-used-in-an-application/6044415#6044415