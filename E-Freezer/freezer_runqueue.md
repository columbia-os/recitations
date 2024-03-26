# Freezer runqueue

> This guide was written by Ryan Wee in Spring 2024. The code snippets and links
> in this post correspond to Linux v5.10.205.

## Introduction

First of all, it is entirely possible (and probable) that there are mistakes in
this guide. If so, feel free to contact the TA team.

This guide is meant to complement [Mitchell's guide](./freezer_sched_class.md)
-- it's a good idea to read both of them. Mitchell's guide explains what each of
the `sched_class` functions do. The aim of this guide is to provide an
event-driven perspective of how these functions are invoked. In particular, how
is the Freezer runqueue modified in response to different events?

## TLDR

- Freezer's `pick_next_task()` implementation does NOT modify the freezer
  runqueue. It only picks the task at the front of the freezer runqueue.
- When a task's timeslice is up, freezer's `task_tick()` implementation is
  responsible for modifying the freezer runqueue.
- When a task voluntarily yields the CPU / finishes its execution, freezer's
  `dequeue_task()` implementation is responsible for modifying the runqueue.

## When a task's timeslice is up

- The timer interrupt code calls [`scheduler_tick()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4005).
- This function invokes [`curr->sched_class->task_tick()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4021).
- Freezer's `task_tick()` implementation should decrement the current task's
  timeslice. (Note that the timeslice should therefore use jiffies as the unit.)
- Freezer's `task_tick()` implementation should move the current task to the end
  of the runqueue if its timeslice is down to 0. At this point, the task is
  still running on the CPU! We've just modified its position on the runqueue.
- Freezer's `task_tick()` implementation should call [`resched_curr()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L608), which sets the
  `TIF_NEED_RESCHED` flag. Again, at this point, the task is still running on the
  CPU!
- The kernel checks `TIF_NEED_RESCHED` on interrupt and userspace return paths.
  When it's safe (e.g. the process isn't holding any spinlocks), the kernel
  calls [`schedule()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4619), which in turn invokes [`__schedule()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4430).
- This eventually calls `pick_next_task()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4516), which calls Freezer's `pick_next_task()` implementation. You can ignore this huge [if block](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4483) for now. You'll see later why it's not invoked in this case.
- Freezer's `pick_next_task()` implementation should return a new task from the
  front of the feezer runqueue.
- `__schedule()` also eventually calls `context_switch()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4548), which is where the actual context switch happens.

## When a task voluntarily sleeps, e.g. because it's waiting for something

- The task calls the [`sched_yield` syscall](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L6120), which in turn invokes ['do_sched_yield()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L6103).
- `do_sched_yield` calls [`current->sched_class->yield_task()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L6111). Freezer's `yield_task()` implementation should set the timeslice of the current task to 0, but this doesn't actually matter in this case (at least, I think it doesn't ...).
- `do_sched_yield`  also calls `schedule`, which in turn calls `__schedule()`.
  Note that `__schedule()` is called with `false` as an argument. See [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4626).
- Recall that the `task_struct` has a [`state` field](https://elixir.bootlin.com/linux/v5.10.205/source/include/linux/sched.h#L653). In this case, the `state` field of
  the `task_struct` is no longer `RUNNABLE`, i.e. it is non-zero. This is
  different from the earlier timer interrupt case, where the `state` field of
  the `task_struct` would still have been `RUNNABLE`, i.e. it would still have
  been zero.
- Since `state` is non-zero and since the `preempt` argument was `false` as
  mentioned above, we go into this [if block](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4483).
- This in turn calls `deactivate_task()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4506).
- `deactivate_task()` calls [`dequeue_task()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L1588), which in turn calls the `dequeue_task` implementation of the current scheduling class.
- Freezer's `dequeue_task()` implementation should remove the current task from
  the freezer runqueue.
- Recall that we're still within an earlier invocation of `__schedule()`.
  This invocation calls `pick_next_task()` and `context_switch()` as mentioned
  above. Since the current task has been removed from the freezer runqueue, this
  causes the next task in the runqueue to be run.

## When a process is done executing

- The process calls the [`exit()` syscall](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/exit.c#L947), which calls [`do_exit()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/exit.c#L760).
- This in turn calls `do_task_dead()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/exit.c#L909), which in turn calls `__schedule()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4565). Note that `__schedule()` is called with `false` as an argument.
- In this case, the `state` field of the `task_struct` is again no longer
  `RUNNABLE`, i.e. it is non-zero. See above for why this eventually calls
  freezer's `dequeue_task()` implementation.
