---
title: "RVOS Exercise Notes IV"
date: 2023-11-12T13:27:28+08:00
draft: true
---

## 15-1

> 参考例子 [code/os/10-swtimer](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/os/10-swtimer)，自己尝试实现一个软件单次触发定时器。参考下面的代码实现标准的创建定时器和删除定时器接口：
> ```c
> /* software timer */ 
> struct timer { 
>   void (*func)(void *arg); 
>   void *arg; 
>   uint32_t timeout_tick; 
> }; 
>  
> struct timer *timer_create( 
>     void (*handler)(void *arg), 
>     void *arg, 
>     uint32_t timeout); 
>  
> void timer_delete(struct timer *timer);
> ```
> - 软件定时器对象列表的组织先采用简单的数组管理方式；
> - 数组方式实验完成后再尝试采用链表的管理方式。为加快超时对象的搜索，要求软件定时器对象在链表中按照超时的前后顺序排序，即实现一个有序的链表。

这里数组管理方式实际上已经默认实现了，参考 [10-swtimer/timer.c](https://github.com/ludics/riscv-operating-system-mooc/blob/exercise/code/os/10-swtimer/timer.c)。贴一下实现的代码：
```c
struct timer {
	void (*func)(void *arg);
	void *arg;
	uint32_t timeout_tick;
};

#define MAX_TIMER 10
static struct timer timer_list[MAX_TIMER];

struct timer *timer_create(void (*handler)(void *arg), void *arg, uint32_t timeout)
{
	/* TBD: params should be checked more, but now we just simplify this */
	if (NULL == handler || 0 == timeout) {
		return NULL;
	}

	/* use lock to protect the shared timer_list between multiple tasks */
	spin_lock();

	struct timer *t = &(timer_list[0]);
	for (int i = 0; i < MAX_TIMER; i++) {
		if (NULL == t->func) {
			break;
		}
		t++;
	}
	if (NULL != t->func) {
		spin_unlock();
		return NULL;
	}

	t->func = handler;
	t->arg = arg;
	t->timeout_tick = _tick + timeout;

	spin_unlock();

	return t;
}

void timer_delete(struct timer *timer)
{
	spin_lock();

	struct timer *t = &(timer_list[0]);
	for (int i = 0; i < MAX_TIMER; i++) {
		if (t == timer) {
			t->func = NULL;
			t->arg = NULL;
			break;
		}
		t++;
	}

	spin_unlock();
}

/* this routine should be called in interrupt context (interrupt is disabled) */
static inline void timer_check()
{
	struct timer *t = &(timer_list[0]);
	for (int i = 0; i < MAX_TIMER; i++) {
		if (NULL != t->func) {
			if (_tick >= t->timeout_tick) {
				t->func(t->arg);

				/* once time, just delete it after timeout */
				t->func = NULL;
				t->arg = NULL;

				break;
			}
		}
		t++;
	}
}
```
可以看到，这里预先定义了一个定时器数组 `timer_list`，在进行 `timer_create`、`timer_delete`、`timer_check` 等操作都需要遍历这个数组。

