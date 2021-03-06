# go-routine调度

##### 参考资料

G-P-M调度结构说明请参考[https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.1.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)

G-P-M调度说明可参考[https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)

[https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

##### Goroutine调度策略

当前M中运行的G进入系统调用、IO、等待channel数据时会发生routine切换

* ##### syscall阻情况下调度

1. 进入系统调用（即调用entersyscall 或者entersyscallblock）
2. 当前M中运行的G进入Gsyscall状态，让出CPU
3. M-&gt;P-&gt;M = null, 即标记当前M attach的P没有关联任何M；如果系统调用时阻塞调用（entersyscallblock），还需要设置M-&gt;P为空，若该P中还有另外的G需要执行，则调度器将该P与某个M绑定执行，否则将P放入全局Pidle队列；调度器将P与某个M绑定运行时，首先检查idle状态的M，如果没有则会创建新的M来执行P
4. 系统调用返回（exitsyscall）时，先检查当前M的P的状态，若不为空且P为Psyscall状态时，则将P-&gt;M设置为当前M，将P的mcache放回到M中，恢复G状态为RUNNING；否之则是从阻塞系统调用中返回，之前该M attach 的P已经完全detach，这时从全局idle P 队列获取P执行，如果没有idle P，则将当前M的G设为GRUNNABLE状态放入全局runq队列，然后停止当前M并调用schedule函数。

> 如果G被阻塞在某个system call操作上，那么不光G会阻塞，执行该G的M也会解绑P\(实质是被sysmon抢走了\)，与G一起进入sleep状态。如果此时有idle的M，则P与其绑定继续执行其他G；如果没有idle M，但仍然有其他G要去执行，那么就会创建一个新M。
>
> 当阻塞在syscall上的G完成syscall调用后，G会去尝试获取一个可用的P，如果没有可用的P，那么G会被标记为runnable，之前的那个sleep的M将再次进入sleep。

* ##### 抢占式调度

系统启动时运行的sysmon会定时（20us～10ms）运行一次检测

```
1. 释放闲置超过5分钟的span物理内存；
2. 如果超过2分钟没有垃圾回收，强制执行；
3. 将长时间未处理的netpoll结果添加到任务队列；
4. 向长时间运行的G任务发出抢占调度；
5. 收回因syscall长时间阻塞的P
```

抢占调度实际上是由retake实现的

```
// forcePreemptNS is the time slice given to a G before it is
// preempted.
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
          ... ...
           // Preempt G if it's running for too long.
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
                continue
            }
            if pd.schedwhen+forcePreemptNS > now {
                continue
            }
            preemptone(_p_)
         ... ...
}
```

由代码可以看出实际的G能独占M的时间为10ms，之将被强占，以防其它G饿死

* ##### channel阻塞或network I/O情况下的调度

如果G被阻塞在某个channel操作或network I/O操作上时，G会被放置到某个wait队列中，而M会尝试运行下一个runnable的G；如果此时没有runnable的G供m运行，那么m将解绑P，并进入sleep状态。当I/O available或channel操作完成，在wait队列中的G会被唤醒，标记为runnable，放入到某P的队列中，绑定一个M继续执行。

