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
static int __cpuinit cpu_callback(struct notifier_block *nfb, unsigned long action, void *hcpu){	int hotcpu = (unsigned long)hcpu;	struct task_struct *p;	switch (action) {	case CPU_UP_PREPARE:	case CPU_UP_PREPARE_FROZEN:		BUG_ON(per_cpu(watchdog_task, hotcpu));
		//创建内核线程,名字是watchdog/X		p = kthread_create(watchdog, hcpu, "watchdog/%d", hotcpu);		if (IS_ERR(p)) {			printk(KERN_ERR "watchdog for %i failed\n", hotcpu);			return NOTIFY_BAD;		}		per_cpu(touch_timestamp, hotcpu) = 0;		per_cpu(watchdog_task, hotcpu) = p;
		//绑定cpu		kthread_bind(p, hotcpu);
	...
}

/* * The watchdog thread - runs every second and touches the timestamp. */static int watchdog(void *__bind_cpu){	struct sched_param param = { .sched_priority = MAX_RT_PRIO-1 };	//设置为先进先出的实时优先级,并设置最高优先级MAX_RT_PRIO-1	sched_setscheduler(current, SCHED_FIFO, &param);	/* initialize timestamp */
	// 获取系统时间, 粗略的取了秒值 cpu_clock(this_cpu) >> 30LL;	__touch_softlockup_watchdog();	set_current_state(TASK_INTERRUPTIBLE);	/*	 * Run briefly once per second to reset the softlockup timestamp.	 * If this gets delayed for more than 60 seconds then the	 * debug-printout triggers in softlockup_tick().	 */	while (!kthread_should_stop()) {		__touch_softlockup_watchdog();		schedule();		if (kthread_should_stop())			break;		set_current_state(TASK_INTERRUPTIBLE);	}	__set_current_state(TASK_RUNNING);	return 0;}
```
此处设置优先级和调度(schedule)相关的说明可参考我的另一篇文章[Linux进程调度](/linux/kernel/2015/07/14/schedule.html)

上面的流程简单来讲就是:

* 每个CPU运行一个 watchdog 内核线程
* 线程设置为实时进程,最高优先级(保证进程切换时能够切换到本进程,如果在特定时间内还没有切换到本进程,可以认为 soft lockup)
* 运行期间更新本CPU的 `touch_timestamp` 变量

但是这样不足以完成检测 soft lockup 的功能, 还需要其他地方的配合, 就是要检查 `touch_timestamp` 变量是否在规定时间内被更新

```
/* * This callback runs from the timer interrupt, and checks * whether the watchdog thread has hung or not: */void softlockup_tick(void){	int this_cpu = smp_processor_id();	unsigned long touch_timestamp = per_cpu(touch_timestamp, this_cpu);	unsigned long print_timestamp;	struct pt_regs *regs = get_irq_regs();	unsigned long now;	/* Is detection switched off? */	if (!per_cpu(watchdog_task, this_cpu) || softlockup_thresh <= 0) {		/* Be sure we don't false trigger if switched back on */		if (touch_timestamp)			per_cpu(touch_timestamp, this_cpu) = 0;		return;	}	if (touch_timestamp == 0) {		__touch_softlockup_watchdog();		return;	}	print_timestamp = per_cpu(print_timestamp, this_cpu);	/* report at most once a second */	if (print_timestamp == touch_timestamp || did_panic)		return;	/* do not print during early bootup: */	if (unlikely(system_state != SYSTEM_RUNNING)) {		__touch_softlockup_watchdog();		return;	}	now = get_timestamp(this_cpu);	/*	 * Wake up the high-prio watchdog task twice per	 * threshold timespan.	 */	if (time_after(now - softlockup_thresh/2, touch_timestamp))		wake_up_process(per_cpu(watchdog_task, this_cpu));	/* Warn about unreasonable delays: */	if (time_before_eq(now - softlockup_thresh, touch_timestamp))		return;	per_cpu(print_timestamp, this_cpu) = touch_timestamp;	spin_lock(&print_lock);	printk(KERN_ERR "BUG: soft lockup - CPU#%d stuck for %lus! [%s:%d]\n",			this_cpu, now - touch_timestamp,			current->comm, task_pid_nr(current));	print_modules();	print_irqtrace_events(current);	if (regs)		show_regs(regs);	else		dump_stack();	spin_unlock(&print_lock);	if (softlockup_panic)		panic("softlockup: hung tasks");}
```

[Softlockup detector and hardlockup detector](https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt)  
[Linux 时钟管理](https://www.ibm.com/developerworks/cn/linux/l-cn-timerm/)  
[高精度定时器（HRTIMER）的原理和实现](http://blog.csdn.net/droidphone/article/details/8074892)  


