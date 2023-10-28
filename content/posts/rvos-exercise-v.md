---
title: "Rvos Exercise Notes V"
date: 2023-09-16T21:43:45+08:00
draft: false
---

## 13-1

> 要求：在 [练习 9-1]({{< relref "/posts/rvos-exercise-iv.md#9-1" >}}) 的基础上进一步改进任务管理功能，增加任务优先级的管理。具体要求：改进 `task_create()`，增加时间片（`timeslice`）参数，具体改进后的函数如下所示：
> ```c
> int task_create(void (*task)(void *param), void *param,
>                 uint8_t priority, uint32_t timeslice);
> ```
> 其中：
> - 其他参数含义不变，见 [练习 9-1]({{< relref "/posts/rvos-exercise-iv.md#9-1" >}}) 的描述；
> - `timeslice`: 任务的时间片大小，单位是操作系统的时钟节拍（tick），此参数指定该任务一次调度可以运行的最大时间⻓度。和 `priotity` 相结合，调度器会首先根据 `priority` 选择优先级最高的任务运行，而 `timeslice` 则决定了当没有更高优先级的任务时，当前正在运行的任务可以运行的最大时间⻓度。

先实现任务的优先级调度，基本将 [练习 9-1]({{< relref "/posts/rvos-exercise-iv.md#9-1" >}}) 的代码 copy 了一下，提交见 [6dda145](https://github.com/ludics/riscv-operating-system-mooc/commit/6dda145acd3217d97b1850409f4e76fa9031845b)。

接下来再实现时间片的功能。

首先，增加 `task_timeslice` 数组，用于保存每个任务的时间片大小：

```c
uint32_t task_timeslice[MAX_TASKS];
```

然后，需要在 `task_create()` 中将 `timeslice` 传递给 `task_timeslice`：

```c
int task_create(void (*task)(void *param), void *param,
                uint8_t priority, uint32_t timeslice)
{
    ...
    task_priorities[task_id] = priority;
    task_timeslice[task_id] = timeslice;
    ...
}
```
之后，需要在 `schedule()` 中实现时间片的功能。首先，需要增加一个 `_cur_timeslice` 变量，表示当前正在运行的任务已经运行的时间片大小：

```c
static int _cur_timeslice = 0;
```
然后，需要在 `schedule()` 中实现时间片的功能：

```c
void schedule(void)
{
    ...
    // 如果当前没有任务在运行，或者当前任务已经退出，则选择下一个任务
	if (_current == -1 || task_status[_current] == TASK_EXITED) {
		_current = next_task_id;
		_cur_timeslice = 1;
	// 检查下个任务的优先级是否比当前任务高，如果高则切换
	// 如果优先级相同，则检查当前任务的时间片是否用完
	} else if (task_priorities[next_task_id] < task_priorities[_current] ||
			(task_priorities[next_task_id] == task_priorities[_current] &&
			 task_timeslice[_current] <= _cur_timeslice )) {
		_current = next_task_id;
		_cur_timeslice = 1;
	} else {
		_cur_timeslice++;
	}
    ...
}
```
这样就完成了修改，代码提交见 [436aa37](https://github.com/ludics/riscv-operating-system-mooc/commit/436aa373f9570fc76aae351e100eb3a8ecb13a34)。

我们再对这个修改进行测试，在 `os_main` 中调用 `task_create` 创建任务时传递 `timeslice` 参数，测试代码如下：

```c
void os_main(void)
{
	task_create(user_task, (void *)3000, 0, 2);
	task_create(user_task, (void *)8003, 3, 2);
	task_create(user_task, (void *)2003, 3, 3);
	task_create(user_task, (void *)5000, 0, 3);
	task_create(user_task, (void *)9003, 3, 4);
	task_create(user_task, (void *)6001, 1, 3);
	task_create(user_task, (void *)7001, 1, 2);
	task_create(user_task, (void *)4000, 0, 4);
}
```
这里我们创建了 8 个任务，其中 3 个优先级为 0，2 个优先级为 1，3 个优先级为 3。其中相同优先级的任务的时间片也都不同，这样可以测试时间片的功能。`make run` 将操作系统跑起来后，运行结果为：
```
Hello, RVOS!
...
Task 3000: Created!
Task 3000: Running...
Task 3000: Running...
Task 3000: Running...
timer interruption!
tick: 1
Task 3000: Running...
Task 3000: Running...
timer interruption!
tick: 2
Task 5000: Created!
Task 5000: Running...
Task 5000: Running...
Task 5000: Running...
timer interruption!
tick: 3
Task 5000: Running...
Task 5000: Running...
timer interruption!
tick: 4
Task 5000: Running...
Task 5000: Running...
timer interruption!
tick: 5
Task 4000: Created!
Task 4000: Running...
Task 4000: Running...
Task 4000: Running...
timer interruption!
tick: 6
Task 4000: Running...
Task 4000: Running...
timer interruption!
tick: 7
Task 4000: Running...
Task 4000: Running...
timer interruption!
tick: 8
Task 4000: Running...
Task 4000: Running...
timer interruption!
tick: 9
Task 3000: Running...
Task 3000: Running...
timer interruption!
tick: 10
Task 3000: Running...
Task 3000: Running...
timer interruption!
tick: 11
Task 5000: Running...
Task 5000: Running...
timer interruption!
tick: 12
Task 5000: Running...
Task 5000: Running...
timer interruption!
tick: 13
Task 5000: Running...
Task 5000: Running...
timer interruption!
tick: 14
Task 4000: Running...
Task 4000: Running...
timer interruption!
tick: 15
...
```
