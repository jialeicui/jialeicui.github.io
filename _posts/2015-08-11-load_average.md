---
layout: post
title:  "load average"
date:   2015-08-11
categories: linux kernel
---

本文中的代码分析基于 linux-2.6.32.67  

## 在哪儿

load average可以在以下的命令输出中找到

* uptime
* w
* top

这些输入中包含的例如 `load average: 0.00, 0.00, 0.08` 的字段就是 load average 了  
三个数字分别代表系统最近1/5/15分钟的负载

## 有什么用

load average是系统负荷的一个参考, 简单来讲, 对于**单核单CPU**的系统, load为 1 表示系统正好能够处理当前所有的任务, 再多就得有任务要等待. 要系统提供正常流畅的服务, 通常需要保持 load 在一个合理的范围

### 多少是合理范围?

#### 基准

当前系统认为有多少个 CPU 的 core, 就认为 load 达到这个值是"满载". 例如:

```
grep -c  "model name" /proc/cpuinfo 
28
```
那么这个系统在 load 为 28 时"满载", 如果把这个 CPU 超线程到 56 个逻辑核, 那么就是 56 满载.

#### 合理范围

大多数系统管理员都使用 70% 这个比例来界定. 当系统的负荷持续达到满载的 70%, 就应该着手查找提高系统负载的原因.


## 系统怎么算的

这里尝试从内核角度来大概看一下系统的 load 是怎么计算出来的.  
首先, 直观的, 这个值计算是在`sched.c` 中进行

```c
void calc_global_load(void)
{
    unsigned long upd = calc_load_update + 10;
    long active;
    // 如果没有到时间, 不进行刷新
    if (time_before(jiffies, upd))
        return;
    //取数据
    active = atomic_long_read(&calc_load_tasks);
    active = active > 0 ? active * FIXED_1 : 0;

    avenrun[0] = calc_load(avenrun[0], EXP_1, active);//1分钟
    avenrun[1] = calc_load(avenrun[1], EXP_5, active);//5分钟
    avenrun[2] = calc_load(avenrun[2], EXP_15, active);//15分钟
    //设定下一次更新的时间
    calc_load_update += LOAD_FREQ;
}
```

`sched.h` 中定义了刷新间隔

```c
#define LOAD_FREQ   (5*HZ+1)    /* 5 sec intervals */
```

这个里面用到的`calc_load_tasks`是经由下面调用栈每1ms进行计算的  

```C
calc_load_tasks ->
scheduler_tick -> 
update_cpu_load -> 
static void calc_load_account_active(struct rq *this_rq)
{
    long nr_active, delta;
    // 从这里可以看出,linux计算load是包含running和uninterruptible的任务
    nr_active = this_rq->nr_running;
    nr_active += (long) this_rq->nr_uninterruptible;

    if (nr_active != this_rq->calc_load_active) {
        delta = nr_active - this_rq->calc_load_active;
        this_rq->calc_load_active = nr_active;
        atomic_long_add(delta, &calc_load_tasks);
    }
}
```
这里进行负载的计算比上面提到的 5s 频繁的多, 实际上这里的计算不只为了输出 load average 的数据, 主要是为 CPU 的负载均衡提供数据, 负载均衡的东西又是一块骨头, 下次再啃.

参考  
[1. Understanding Linux CPU Load](http://blog.scoutapp.com/articles/2009/07/31/understanding-load-averages)  
[2. UNIX Load Average Part 1:How It Works](http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf)  
[3. Unix/Linux 的 Load 初级解释](http://dbanotes.net/arch/unix_linux_load.html)  
[4. 系统管理员工具包: 监视运行缓慢的系统](http://www.ibm.com/developerworks/cn/aix/library/au-satslowsys.html)  
[5. 理解Linux系统负荷](http://www.ruanyifeng.com/blog/2011/07/linux_load_average_explained.html)  
[6. Load computing](https://en.wikipedia.org/wiki/Load_(computing))  

