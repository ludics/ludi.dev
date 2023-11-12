---
title: "RVOS Exercise Notes III"
date: 2023-09-10T22:35:46+08:00
draft: false
---

## 9-1

> 要求：参考 [`code/os/04-multitask`](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/os/04-multitask)，在此基础上进一步改进任务管理功能。具体要求：
> - 改进 `task_create()`，提供更多的参数，具体改进后的函数如下所示：
> ```c
> int task_create(void (*task) (void *param), void *param, uint8_t priority);
> ```
> 其中，`param` 用于在创建任务执行函数时可带入参数，如果没有参数则传入 `NULL` 即可；`priority` 用于指定任务的优先级，目前要求最多支持 256 级，0 最高，依此类推。同时修改任务调度算法，在原先简单轮转的基础上支持按照优先级排序，优先选择优先级高的任务执行，如果优先级相同则采用简单轮转的方式。
> - 增加任务退出接口 `task_exit()`，当前任务可以通过调用该接口退出执行，内核负责将该任务回收，并调度下一个可运行任务。建议的接口函数如下：
> ```c
> void task_exit(void);
> ```

### 代码分析

首先看 [`code/os/03-contextswitch`](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/os/03-contextswitch)。

上下文结构体 `struct context` 中保存着初 `x0` 外的 31 个寄存器的值。OS 初始化时，会调用 `sched_init()` 函数，首先将 `mscratch` 寄存器设置为 0，然后将用户任务 `user_task0` 的地址保存到 `ctx_task.ra`，将任务栈地址保存到 `ctx_task.sp`。`schedule` 函数执行具体的调度，这个函数会调用上下文切换的核心函数 `switch_to`，从而切换上下文，进而实现任务切换。下面具体分析 `switch_to` 函数。

`switch_to` 函数采用汇编语言编写。首先交换 `t6` 寄存器与 `mscratch` 寄存器的值，`mscratch` 寄存器是用来保存当前任务的 `context` 地址的。如果此值为 0，表示是第一个任务，直接保存要调度的下个任务的 `context` 地址到 `mscratch` 寄存器中，同时用 `reg_store` 宏将下个任务的 `context` 中的值恢复到寄存器中。如果此值不为 0，先调用 `reg_save` 宏将当前的寄存器值保存到当前任务的 `context` 中，注意我们使用 `t6` 寄存器，所有保存完其他寄存器后，还要从 `mscratch` 中读出本来的寄存器值，保存到 `context` 的对应位置；保存完当前寄存器之后，再从下一个任务的 `context` 恢复寄存器，从而实现任务切换。

[`code/os/03-contextswitch`](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/os/03-contextswitch) 只实现了切换到一个用户任务中，[`code/os/04-multitask`](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/os/04-multitask) 通过在用户任务函数中调用 `task_yield` 函数实现了多个用户任务的切换。

### 练习结果

练习的代码地址为 [ex_9_1](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/exercises/ex_9_1)，其中对应修改的 commit 为 [4e9f64875](https://github.com/ludics/riscv-operating-system-mooc/commit/4e9f64875f9c3bc6094a13794211695a84f236e9)。

为了支持优先级调度与任务退出，新增了一些数据结构，如下所示：
```c
uint8_t task_priorities[MAX_TASKS];

#define TASK_EMPTY 0
#define TASK_READY 1
#define TASK_RUNNING 2
#define TASK_BLOCKED 3
#define TASK_EXITED 4

uint8_t task_status[MAX_TASKS];

#define task_schedulable(task_id) (task_status[task_id] == TASK_READY || task_status[task_id] == TASK_RUNNING)
```
其中 `task_priorities` 数组用于保存每个任务的优先级，`task_status` 数组用于保存每个任务的状态，`task_schedulable` 宏用于判断任务是否可调度。

`task_create` 函数的实现如下所示：
```c
int task_create(void (*start_routin)(void* param), void* param, uint8_t priority)
{
  int task_id = -1;
  for (int i = 0; i < MAX_TASKS; i++) {
    if (task_status[i] == TASK_EMPTY || task_status[i] == TASK_EXITED) {
      task_id = i;
      break;
    }
  }
  if (task_id == -1) {
    return -1;
  }
  task_status[task_id] = TASK_READY;
  task_priorities[task_id] = priority;
  ctx_tasks[task_id].sp = (reg_t) &task_stack[task_id][STACK_SIZE];
  ctx_tasks[task_id].ra = (reg_t) start_routin;
  if (param != NULL) {
    ctx_tasks[task_id].a0 = (reg_t) param;
  }
  return 0;
}
```

`task_exit` 函数的实现如下所示：
```c
void task_exit(void)
{
  task_status[_current] = TASK_EXITED;
  schedule();
}
```

`schedule` 函数的实现如下所示：
```c
void schedule()
{
  uint8_t priority = 0xff;
  for (int i = 0; i < MAX_TASKS; i++) {
    if (task_schedulable(i) && task_priorities[i] < priority) {
      priority = task_priorities[i];
    }
  }
  int next_task_id = -1;
  for (int i = 0; i < MAX_TASKS; i++) {
    if (task_schedulable(i) && task_priorities[i] == priority) {
      if (i > _current) {
        next_task_id = i;
        break;
      }
    }
  }
  if (next_task_id == -1) {
    for (int i = 0; i < MAX_TASKS; i++) {
      if (task_schedulable(i) && task_priorities[i] == priority) {
        next_task_id = i;
        break;
      }
    }
  }
  if (next_task_id == -1) {
    panic("no schedulable task");
    return;
  }
  if (next_task_id == _current) {
    task_status[_current] = TASK_RUNNING;
    return;
  }
  _current = next_task_id;
  struct context* next = &(ctx_tasks[_current]);
  task_status[_current] = TASK_RUNNING;
  switch_to(next);
}
```

使用如下函数进行测试：
```c
void user_task(void* param) {
  int task_id = (int)param;
  printf("Task %d: Created!\n", task_id);
  int iter_cnt = task_id;
  while (1) {
    printf("Task %d: Running...\n", task_id);
    task_delay(DELAY);
    task_yield();
    if (iter_cnt-- == 0) {
      break;
    }
  }
  printf("Task %d: Finished!\n", task_id);
  task_exit();
}

/* NOTICE: DON'T LOOP INFINITELY IN main() */
void os_main(void)
{
  task_create(user_task1, (NULL), 255);
  task_create(user_task, (void *)3, 0);
  task_create(user_task, (void *)4, 0);
  task_create(user_task, (void *)5, 1);
}
```

运行结果如下所示：
```
Task 3: Created!
Task 3: Running...
Task 4: Created!
Task 4: Running...
Task 3: Running...
Task 4: Running...
Task 3: Running...
Task 4: Running...
Task 3: Running...
Task 4: Running...
Task 3: Finished!
Task 4: Running...
Task 4: Finished!
Task 5: Created!
Task 5: Running...
Task 5: Running...
Task 5: Running...
Task 5: Running...
Task 5: Running...
Task 5: Running...
Task 5: Finished!
Task 1: Created!
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
......
```

## 9-2

> 目前 [`code/os/04-multitask`](https://github.com/ludics/riscv-operating-system-mooc/tree/exercise/code/os/04-multitask) 实现的任务调度中，前一个用户任务直接调用 `task_yield()` 函数并最终调用`switch_to()` 切换到下一个用户任务。`task_yield()` 作为内核路径借用了用户任务的栈，当用户任务的函数调用层次过多或者 task_yield() 本身函数内部继续调用函数，可能会导致用户任务的栈空间溢出。参考 "[mini-riscv-os](https://github.com/cccriscv/mini-riscv-os)" 的 [03-MultiTasking](https://github.com/cccriscv/mini-riscv-os/tree/master/03-MultiTasking) 的实现，为内核调度单独实现一个任务，在任务切换中，前一个用户任务首先切换到内核调度任务，然后再由内核调度任务切换到下一个用户任务，这样就可以避免前面提到的问题了。
> 要求：参考以上设计，并尝试实现之。

具体实现参考 [61fe623a225c](https://github.com/ludics/riscv-operating-system-mooc/commit/61fe623a225c483669d386a1a0749708a1164d8a)。

首先在 `start_kernel()` 函数中，将
```c
os_main();
schedule();
```
修改为
```c
os_main();
while (1) {
  schedule();
}
```

创建一个内核上下文结构 `ctx_os`，并在 `sched_init()` 中将 `mscratch` 寄存器设置为 `ctx_os` 的地址：
```c
struct context ctx_os;

void sched_init(void)
{
  mscratch_write((reg_t) &ctx_os);
}
```
这样在 `start_kernel()` 第一次调用 `schedule()` 时，会切换到用户任务上下文，同时将当前的任务上下文保存到 `ctx_os` 中。

利用 `back_to_os()` 函数实现从用户任务切换到内核任务：
```c
void back_to_os(void)
{
  struct context* next = &(ctx_os);
  switch_to(next);
}
```
最后，修改 `task_yield()` 与 `task_exit()` 函数，使得它们调用 `back_to_os()` 函数：
```c
void task_yield(void)
{
  task_status[_current] = TASK_READY;
  back_to_os();
}

void task_exit(void)
{
  task_status[_current] = TASK_EXITED;
  back_to_os();
}
```

这样，用户任务切换的流程就变成了：
```
user_task -> task_yield -> back_to_os -> switch_to(ctx_os) -> schedule -> switch_to(ctx_user) -> user_task
```
而之前的流程是：
```
user_task -> task_yield -> schedule -> switch_to(ctx_user) -> user_task
```

## 13-1

> 要求：在 [练习 9-1]({{< relref "/posts/rvos-exercise-iii.md#9-1" >}}) 的基础上进一步改进任务管理功能，增加任务优先级的管理。具体要求：改进 `task_create()`，增加时间片（`timeslice`）参数，具体改进后的函数如下所示：
> ```c
> int task_create(void (*task)(void *param), void *param,
>                 uint8_t priority, uint32_t timeslice);
> ```
> 其中：
> - 其他参数含义不变，见 [练习 9-1]({{< relref "/posts/rvos-exercise-iii.md#9-1" >}}) 的描述；
> - `timeslice`: 任务的时间片大小，单位是操作系统的时钟节拍（tick），此参数指定该任务一次调度可以运行的最大时间⻓度。和 `priotity` 相结合，调度器会首先根据 `priority` 选择优先级最高的任务运行，而 `timeslice` 则决定了当没有更高优先级的任务时，当前正在运行的任务可以运行的最大时间⻓度。

先实现任务的优先级调度，基本将 [练习 9-1]({{< relref "/posts/rvos-exercise-iii.md#9-1" >}}) 的代码 copy 了一下，提交见 [6dda145](https://github.com/ludics/riscv-operating-system-mooc/commit/6dda145acd3217d97b1850409f4e76fa9031845b)。

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
