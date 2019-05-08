# go-routine调度

G-P-M调度结构说明请参考[https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.1.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)

当前M中运行的G进入系统调用或者等待channel数据时会发生routine切换，下面以进入系统调用举例

1. 进入系统调用（即调用entersyscall 或者entersyscallblock）
2. 当前M中运行的G进入Gsyscall状态，让出CPU
3. M-&gt;P-&gt;M = null, 即标记当前M attach的P没有关联任何M；如果系统调用时阻塞调用（entersyscallblock），还需要设置M-&gt;P为空，若该P中还有另外的G需要执行，则调度器将该P与某个M绑定执行，否则将P放入全局Pidle队列
4. 系统调用返回（exitsyscall）时，先检查当前M的P的状态，若不为空且P为Psyscall状态时，则将P-&gt;M设置为当前M，将P的mcache放回到M中，恢复G状态为RUNNING；否之则是从阻塞系统调用中返回，之前该M attach 的P已经完全detach，这时从全局idle P 队列获取P执行，如果没有idle P，则将M放入全局idle M队列中。



