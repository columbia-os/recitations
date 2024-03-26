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
is the freezer runqueue modified in response to different events?

## TLDR

- Freezer's `pick_next_task()` implementation does NOT modify the freezer
  runqueue. It only picks the task at the front of the freezer runqueue.
- When a task is preempted because its timeslice is up, freezer's `task_tick()`
  implementation is responsible for modifying the freezer runqueue.
- When a task voluntarily yields the CPU, there are two possibilities:
    - The task's state is `RUNNABLE`, i.e. it would like to stay on the
        run queue: In this case, freezer's `yield_task()` and `task_tick()`
        implementations are responsible for modifying the freezer runqueue.
    - The task's state is NOT `RUNNABLE`, i.e. it would like to leave the
        run queue: In this case, freezer's `dequeue_task()` implementation is responsible for modifying the freezer runqueue.

## When a task's timeslice is up

- The timer interrupt code calls [`scheduler_tick()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4005).
- This function invokes [`curr->sched_class->task_tick()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4021).
- Freezer's `task_tick()` implementation should decrement the current task's
  timeslice. (Note that the timeslice should therefore use jiffies as the unit.)
- If the current task's timeslice is down to 0:
    - Freezer's `task_tick()` implementation should move the current task to the
      end of the runqueue. At this point, the task is still running on the CPU!
      We've just modified its position on the runqueue.
    - Freezer's `task_tick()` implementation should call [`resched_curr()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L608), which
      sets the `TIF_NEED_RESCHED` flag. Again, at this point, the task is still running on the CPU!
- The kernel checks `TIF_NEED_RESCHED` on interrupt and userspace return paths.
  When it's safe (e.g. the process isn't holding any spinlocks), the kernel
  calls [`schedule()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4619), which in turn invokes [`__schedule()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4430).
- This huge [`if` block](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4483) is NOT executed.
    - In particular, recall that the `task_struct` has a [`state` field](https://elixir.bootlin.com/linux/v5.10.205/source/include/linux/sched.h#L653). When a task is `RUNNABLE`, the state field is zero. When a task is not `RUNNABLE`, the state field is non-zero.
    - In this case, the task is still `RUNNABLE`, i.e. `prev_state` is zero. So the `if` block is not executed.
- `schedule()` eventually calls `pick_next_task()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4516), which calls Freezer's `pick_next_task()` implementation.
- Freezer's `pick_next_task()` implementation should return a new task from the
  front of the feezer runqueue.
- `__schedule()` also eventually calls `context_switch()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4548), which is where the actual context switch happens.

## When a task voluntarily yields, but is RUNNABLE

In this case, the task wants to be remain on the runqueue. It's just telling
the kernel: "Okay, you can shift me to the back of the runqueue because I'm
nice. But feel free to select me again when you want to!"

- The task calls the [`sched_yield` syscall](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L6120), which in turn invokes [`do_sched_yield()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L6103).
- `do_sched_yield` calls [`current->sched_class->yield_task()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L6111).
- Freezer's `yield_task()` implementation should set the timeslice of the
  task to 0.
- Now the next time we have a timer interrupt, the task is 'pre-empted' as described above by freezer's `task_tick()` implementation. (I put that in quotation marks because the task voluntarily decided to let itself be pre-empted.)

## When a task voluntarily yields, and is not RUNNABLE

In this case, the task wants to be taken off the runqueue. This could be because the task called `__wait_event_interruptible()`, or `sleep()`, or anything that basically indicates it wants to suspend its execution for the near future. Another common case would be when the task finishes its execution.

- The task starts by modifying its state so that it is no longer `RUNNABLE`. It then calls `__schedule()` with `false` as an argument.
    - In the case of a task finishing its execution:
        - The task calls the [`exit()` syscall](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/exit.c#L947), which calls [`do_exit()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/exit.c#L760).
        - This in turn calls `do_task_dead()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/exit.c#L909).
        - `do_task_dead()` modifies the task state [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4560), and then calls `__schedule()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4565).
    - In the case of a task calling `sleep()`:
        - The task calls the [`nanosleep()` syscall](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/time/hrtimer.c#L2016), which calls `hrtimer_nanosleep()` and then [`do_nanosleep()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/time/hrtimer.c#L1933).
        - `do_nanosleep()` modifies the task state [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/time/hrtimer.c#L1938), and then calls `freezable_schedule()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/time/hrtimer.c#L1942). This in turn calls `schedule()`.
- In all of these cases, when `__schedule()` is called, the `state` field of the `task_struct` is no longer `RUNNABLE`, i.e. it is non-zero. Since `state` is non-zero and since the `preempt` argument was `false` as mentioned above, we go into this [`if` block](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4483).
- This in turn calls `deactivate_task()` [here](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L4506).
- `deactivate_task()` calls [`dequeue_task()`](https://elixir.bootlin.com/linux/v5.10.205/source/kernel/sched/core.c#L1588), which in turn calls the `dequeue_task()` implementation of the current scheduling class.
- Freezer's `dequeue_task()` implementation should remove the current task from the freezer runqueue.
- Recall that we're still within an earlier invocation of `__schedule()`. This invocation calls `pick_next_task()` and `context_switch()` as mentioned above. Since the current task has been removed from the freezer runqueue, this causes the next task in the runqueue to be run.
