
当vector指令执行过成中发生trap, 当前`*epc` csr需要指向发生trap的这条vector指令，`vstart`需要指向发生异常的element index

# 1. Precise Vector Traps

* 所有在发生vector指令trap之前执行的指令都已经commit
* 所有在发生vector指令trap之后执行的指令都不能修改architectural state
* 在这条向量指令中，在引起trap的element之前的element都已经commit
* 在这条向量指令中，在引起trap的element及以后的element都不能修改architectural state


# 2. Imprecise Vector Traps

* 在发生vector指令trap之前执行的指令可能还没有commit
* 在发生vector指令trap之后执行的指令可能已经修改了architectural state

这适用于报告error或者终止程序执行的情形
当前spec不支持imprecise traps

# 3. Swappable traps

这是另外一种trap mode
当trap发生的时候，一些特殊的指令可以保存或者恢复vector unit的microarchitectural states，使得程序可以正常执行下去 （在imprecise trap下)

目前并没有相关定义
