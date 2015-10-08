---
layout: post
title:  soft lockup
date:   2015-10-08
categories: linux kernel
---
本文基于Linux 2.6.32.67

###什么是soft lockup

>A 'softlockup' is defined as a bug that causes the kernel to loop in
kernel mode for more than 20 seconds (see "Implementation" below for
details), without giving other tasks a chance to run.

###实现
检测soft lockup的实现在 `kernel/softlockup.c` 中

```
// early_initcall ->
// spawn_softlockup_task ->
static int __cpuinit cpu_callback(struct notifier_block *nfb, unsigned long action, void *hcpu)
		//创建内核线程,名字是watchdog/X
		//绑定cpu
	...
}

/*
	// 获取系统时间, 粗略的取了秒值 cpu_clock(this_cpu) >> 30LL;
```
此处设置优先级和调度(schedule)相关的说明可参考我的另一篇文章[Linux进程调度](/linux/kernel/2015/07/14/schedule.html)

上面的流程简单来讲就是:

* 每个CPU运行一个 watchdog 内核线程
* 线程设置为实时进程,最高优先级(保证进程切换时能够切换到本进程,如果在特定时间内还没有切换到本进程,可以认为 soft lockup)
* 运行期间更新本CPU的 `touch_timestamp` 变量

但是这样不足以完成检测 soft lockup 的功能, 还需要其他地方的配合, 就是要检查 `touch_timestamp` 变量是否在规定时间内被更新

```
/*
```

[Softlockup detector and hardlockup detector](https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt)  
[Linux 时钟管理](https://www.ibm.com/developerworks/cn/linux/l-cn-timerm/)  
[高精度定时器（HRTIMER）的原理和实现](http://blog.csdn.net/droidphone/article/details/8074892)  

