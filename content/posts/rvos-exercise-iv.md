---
title: "Rvos Exercise Notes IV"
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
