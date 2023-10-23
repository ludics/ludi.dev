---
title: "Rvos Exercise V"
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

理解：当 `timeslice` 为 0 时，表示该任务没有时间片限制，可以一直运行，直到被更高优先级的任务抢占；当 `timeslice` 不为 0 时，表示该任务的运行时间是有限制的；当运行时间超过 `timeslice` 时，调度器会强制把该任务切换出去，让其他任务运行。

先实现任务的优先级调度，基本将 [练习 9-1]({{< relref "/posts/rvos-exercise-iv.md#9-1" >}}) 的代码 copy 了一下，提交见 [6dda145](https://github.com/ludics/riscv-operating-system-mooc/commit/6dda145acd3217d97b1850409f4e76fa9031845b)。

接下来再实现时间片的功能。

