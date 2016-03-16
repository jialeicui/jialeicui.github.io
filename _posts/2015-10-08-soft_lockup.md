---
layout: post
title:  Soft Lockup
date:   2015-10-08
categories: linux kernel
---
本文基于Linux 2.6.32.67

### 什么是soft lockup

>A 'softlockup' is defined as a bug that causes the kernel to loop in
kernel mode for more than 20 seconds (see "Implementation" below for
details), without giving other tasks a chance to run.

soft lockup 出现时, 一般能从远端 ping 通主机, 但是命令行全都无效, 基本无法工作.

### 检测原理

检测soft lockup的实现在 `kernel/softlockup.c` 中

```c
// early_initcall ->
// spawn_softlockup_task ->
static int __cpuinit cpu_callback(struct notifier_block *nfb, unsigned long action, void *hcpu)
{
	int hotcpu = (unsigned long)hcpu;
	struct task_struct *p;

	switch (action) {
	case CPU_UP_PREPARE:
	case CPU_UP_PREPARE_FROZEN:
		BUG_ON(per_cpu(watchdog_task, hotcpu));
		//创建内核线程,名字是watchdog/X
		p = kthread_create(watchdog, hcpu, "watchdog/%d", hotcpu);
		if (IS_ERR(p)) {
			printk(KERN_ERR "watchdog for %i failed\n", hotcpu);
			return NOTIFY_BAD;
		}
		per_cpu(touch_timestamp, hotcpu) = 0;
		per_cpu(watchdog_task, hotcpu) = p;
		//绑定cpu
		kthread_bind(p, hotcpu);
	...
}

/*
 * The watchdog thread - runs every second and touches the timestamp.
 */
static int watchdog(void *__bind_cpu)
{
	struct sched_param param = { .sched_priority = MAX_RT_PRIO-1 };
	//设置为先进先出的实时优先级,并设置最高优先级MAX_RT_PRIO-1
	sched_setscheduler(current, SCHED_FIFO, &param);

	/* initialize timestamp */
	// 获取系统时间, 粗略的取了秒值 cpu_clock(this_cpu) >> 30LL;
	__touch_softlockup_watchdog();

	set_current_state(TASK_INTERRUPTIBLE);
	/*
	 * Run briefly once per second to reset the softlockup timestamp.
	 * If this gets delayed for more than 60 seconds then the
	 * debug-printout triggers in softlockup_tick().
	 */
	while (!kthread_should_stop()) {
		__touch_softlockup_watchdog();
		schedule();

		if (kthread_should_stop())
			break;

		set_current_state(TASK_INTERRUPTIBLE);
	}
	__set_current_state(TASK_RUNNING);

	return 0;
}
```
此处设置优先级和调度(schedule)相关的说明可参考我的另一篇文章[Linux进程调度](/blog/schedule.html)

上面的流程简单来讲就是:

* 每个CPU运行一个 watchdog 内核线程
* 线程设置为实时进程,最高优先级(保证进程切换时能够切换到本进程,如果在特定时间内还没有切换到本进程,可以认为 soft lockup)
* 运行期间更新本CPU的 `touch_timestamp` 变量

但是这样不足以完成检测 soft lockup 的功能, 还需要其他地方的配合, 就是要检查 `touch_timestamp` 变量是否在规定时间内被更新

```c
// 此函数会被 hrtimer 的中断定期调用
/*
 * This callback runs from the timer interrupt, and checks
 * whether the watchdog thread has hung or not:
 */
void softlockup_tick(void)
{
	int this_cpu = smp_processor_id();
	unsigned long touch_timestamp = per_cpu(touch_timestamp, this_cpu);
	unsigned long print_timestamp;
	struct pt_regs *regs = get_irq_regs();
	unsigned long now;

	/* Is detection switched off? */
	if (!per_cpu(watchdog_task, this_cpu) || softlockup_thresh <= 0) {
		/* Be sure we don't false trigger if switched back on */
		if (touch_timestamp)
			per_cpu(touch_timestamp, this_cpu) = 0;
		return;
	}

	if (touch_timestamp == 0) {
		__touch_softlockup_watchdog();
		return;
	}

	print_timestamp = per_cpu(print_timestamp, this_cpu);

	/* report at most once a second */
	if (print_timestamp == touch_timestamp || did_panic)
		return;

	/* do not print during early bootup: */
	if (unlikely(system_state != SYSTEM_RUNNING)) {
		__touch_softlockup_watchdog();
		return;
	}

	now = get_timestamp(this_cpu);

	/*
	 * Wake up the high-prio watchdog task twice per
	 * threshold timespan.
	 */
	if (time_after(now - softlockup_thresh/2, touch_timestamp))
		wake_up_process(per_cpu(watchdog_task, this_cpu));

	/* Warn about unreasonable delays: */
	if (time_before_eq(now - softlockup_thresh, touch_timestamp))
		return;

	per_cpu(print_timestamp, this_cpu) = touch_timestamp;

	spin_lock(&print_lock);
	printk(KERN_ERR "BUG: soft lockup - CPU#%d stuck for %lus! [%s:%d]\n",
			this_cpu, now - touch_timestamp,
			current->comm, task_pid_nr(current));
	print_modules();
	print_irqtrace_events(current);
	if (regs)
		show_regs(regs);
	else
		dump_stack();
	spin_unlock(&print_lock);

	if (softlockup_panic)
		panic("softlockup: hung tasks");
}
```
可以看出这段代码的主要工作:

* 每0.5个 softlockup\_thresh 间隔唤醒 watchdog 一次,使其刷新 per\_cpu 的 `touch_timestamp`
* 检查当前 CPU 时间和 `touch_timestamp` 是否超过 softlockup\_thresh, 如果超过, 则打印信息或者 panic

### 如何调试

出现 soft lockup 后, 在系统信息里会有类似下面的信息:

```sh
Pid: 27196, comm: kni_single Not tainted 2.6.32-358.el6.x86_64 #1 IBM IBM System x3550 M4 Server -[7914O9E]-/00Y8603
RIP: 0010:[<ffffffff8150ffce>]  [<ffffffff8150ffce>] _spin_lock+0x1e/0x30
RSP: 0018:ffff880287543e10  EFLAGS: 00000297
RAX: 000000000000d109 RBX: ffff880287543e10 RCX: 0000000000000000
RDX: 000000000000d108 RSI: dead000000200200 RDI: ffff8803298f7188
RBP: ffffffff8100bb93 R08: ffff88028754e0e0 R09: 001a898187506c56
R10: 0000000000000001 R11: 0000000000000000 R12: ffff880287543d90
R13: ffff8803298f7188 R14: ffff8803298f7140 R15: ffffffff81516d7b
FS:  0000000000000000(0000) GS:ffff880287540000(0000) knlGS:0000000000000000
CS:  0010 DS: 0018 ES: 0018 CR0: 000000008005003b
CR2: 00000000006d4660 CR3: 0000000001a85000 CR4: 00000000000407e0
DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
Process kni_single (pid: 27196, threadinfo ffff88027b7ba000, task ffff88027b69eaa0)
Stack:
ffff880287543e40 ffffffff8149d08f ffffffff8105ad3e ffff88047ca44000
<d> ffff88047ca44000 ffff8803298f72d0 ffff880287543ed0 ffffffff81081817
<d> ffffffff810a7fd0 ffff88047ca45c20 ffff88047ca45820 ffff88047ca45420
Call Trace:
<IRQ> 
[<ffffffff8149d08f>] ? tcp_keepalive_timer+0x1f/0x280
[<ffffffff8105ad3e>] ? scheduler_tick+0x11e/0x260
[<ffffffff81081817>] ? run_timer_softirq+0x197/0x340
[<ffffffff810a7fd0>] ? tick_sched_timer+0x0/0xc0
[<ffffffff8102e90d>] ? lapic_next_event+0x1d/0x30
[<ffffffff81076fb1>] ? __do_softirq+0xc1/0x1e0
[<ffffffff8109b75b>] ? hrtimer_interrupt+0x14b/0x260
[<ffffffff8100c1cc>] ? call_softirq+0x1c/0x30
[<ffffffff8100de05>] ? do_softirq+0x65/0xa0
[<ffffffff81076d95>] ? irq_exit+0x85/0x90
[<ffffffff81516d80>] ? smp_apic_timer_interrupt+0x70/0x9b
[<ffffffff8100bb93>] ? apic_timer_interrupt+0x13/0x20
```

### 小技巧

* `softlockup_thresh`  
/proc/sys/kernel/softlockup\_thresh 用于配置前面提到的阈值

* `softlockup_panic`  
因为自己程序的问题导致系统 soft lockup, 如果这时候是 ssh 连接的, soft lockup 出现时, 命令行全都不响应, 需要用远程管理卡或者手动重启  
这时可以配置 /proc/sys/kernel/softlockup\_panic 为 1, soft lockup 时系统就会自动重启了 (当然, 这依赖于 /proc/sys/kernel/panic 不为0)

[Softlockup detector and hardlockup detector](https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt)  
[Linux 时钟管理](https://www.ibm.com/developerworks/cn/linux/l-cn-timerm/)  
[高精度定时器（HRTIMER）的原理和实现](http://blog.csdn.net/droidphone/article/details/8074892)  
[Linux Crash debug tips - I have a soft lockup, what is causing it ?](http://damntechnology.blogspot.co.uk/2010/04/linux-crash-debug-tips-i-have-soft.html)  


