# go-routine调度

G-P-M调度结构说明请参考[https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.1.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)

当前M中运行的G进入系统调用或者等待channel数据时会发生routine切换，下面以进入系统调用举例

1. 进入系统调用（即调用entersyscall 或者entersyscallblock）
2. 当前M中运行的G进入Psyscall状态，让出CPU
3. M-&gt;P-&gt;M = null, 即标记当前M attach的P没有关联任何M；如果系统调用时阻塞调用（entersyscallblock），还需要设置M-&gt;P为空



