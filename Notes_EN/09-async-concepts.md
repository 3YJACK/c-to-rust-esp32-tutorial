# Preface

Through the previous chapters, we have gained a certain understanding and practical experience of the basic Rust embedded development process. Even for peripherals we have not yet covered, I believe readers can extrapolate from what they have learned and develop by referring to the official documentation.

Therefore, what we will learn next is Rust's concurrency model — **async programming**.

For multitasking scenarios, in traditional embedded development the common solutions are RTOS or Linux, which implement concurrent multitasking scheduling through the system kernel. For embedded concurrency scenarios in Rust, although an RTOS approach can also be used, the language's own feature `async/await` provides a **lighter, more efficient, and more modern** solution for embedded concurrency — one that is also more highly regarded in the Rust community — namely async concurrency.

# What is async

Simply put, **async means "non-blocking waiting"**. In synchronous programming, when you call a time-consuming function — for example, a 1000 ms delay — the CPU sits there idling until the 1000 ms are over before executing the next line of code. In async programming, when calling the same kind of time-consuming operation, the CPU does not wait foolishly; instead, it **immediately switches to executing other tasks**, and only when the delay ends does it come back to continue executing the following code.

In Rust's native language features:

- `async fn` defines a task function that can be suspended and resumed.

- `.await` is a suspension point that voluntarily yields the CPU. If the current operation is not yet ready, the task voluntarily yields the CPU to go execute other tasks.

# Why async

Why does Rust choose async instead of a traditional RTOS?

- **Native language support**: The Rust language itself has built-in `async/await` async syntax. The async model naturally fits with Rust's ownership system, lifetimes, and other features, enabling compile-time memory and thread safety checks — this is almost the "officially designated" concurrency solution.

- **Extreme resource efficiency**: Tasks in a traditional RTOS all require a fixed-size stack to be allocated, and task switching requires kernel intervention to save and restore context, which is costly. Rust's async tasks are **stackless**: they are compiled into a state machine at compile time, storing only the local variables at the current `await` point. Therefore memory overhead can be extremely small, and multiple async tasks can **share the same system stack**, with no kernel involvement in task switching. The lightweight nature of async allows it to support far more concurrent units than RTOS tasks.

```rust
// The code you write:
async fn example(x: u32) -> u32 {
    let a = read_sensor().await;  // State 0 → 1
    let b = a + x;                // State 1
    let c = send_data(b).await;   // State 1 → 2
    c + 1                         // State 2
    ......                        
}

// The compiler generates a structure like this:
enum ExampleStateMachine {
    State0 { x: u32 },
    State1 { x: u32, a: u32 },
    State2 { x: u32, a: u32, b: u32 },
    ......
    Done,
}
```

- **Modern development experience**: Compared with manually writing complex state machines or interrupt handlers, `async/await` lets developers **write concurrent logic in an almost synchronous coding style**.

# How it works

**Rust's async mechanism consists of tasks (Future), wakers (Waker), and executors (Executor).** A task itself decides when to pause and resume; the waker is responsible for notifying the executor when a task is ready; and the executor is responsible for scheduling other tasks to run when a task is suspended.

1. **The `Future` trait and `async`/`.await`**

All async operations begin with the `Future` trait, which represents **a value whose computation will complete and be returned at some point in the future**. `poll` is the core method of the `Future` trait; the executor calls it to advance the task and to obtain the task's execution status. `poll` returns:

- `Poll::Ready(value)`: the task is complete and returns the result `value`.

- `Poll::Pending`: the task cannot complete for now and needs to wait for some event.

Implementing `Future` manually is rather tedious. Therefore, Rust provides the more convenient `async`/`.await` syntactic sugar, making async code as intuitive to write as synchronous code.

- `async`: when you write `async fn` before a function, the compiler converts it into a **state machine** that implements the `Future` trait. Each `.await` point is a state of this state machine.

- `.await`: at an `.await` point, if the `Future` returns `Poll::Pending`, the current task is suspended and hands over control; once the `Future` is ready, it is awakened by the `Waker` and polled again.
2. **`Waker` and event-driven execution**

`Waker` is responsible for "waking" a task when it can continue executing. When a `Future` returns `Pending`, it registers the `Waker` with some event source (such as a timer or a serial port interrupt). When that event occurs, the event source calls the `Waker`'s `wake()` method. Once `wake()` is called, the executor knows that this `Future` can be `poll`ed again. This avoids the executor meaninglessly polling all tasks and achieves **event-driven** execution.

3. **`Executor` and task scheduling**

`Executor` is an executor responsible for managing and driving the execution of `Future`s. A typical executor loops through the following operations:

1. Find a runnable `Future` (for example, one just awakened by a `Waker`).

2. Call its `poll` method.

3. If it returns `Pending`, put it back in the queue to wait for the next wake-up.

4. If it returns `Ready`, mark it as complete.

Note that **interrupt handlers do not support the `Future` trait**. On one hand, there is no executor inside an interrupt service routine to drive a `Future`; on the other hand, an interrupt service routine itself cannot be "suspended" and therefore cannot perform `.await`. Consequently, **async operations that require `.await` or an executor to drive them cannot be used there — only their synchronous versions can be used**.

# Mainstream async frameworks

From how async programming is implemented in Rust, we can see that the executor is the key to implementing concurrent multitasking scheduling. But Rust itself does not implement an executor; instead, it chooses to leave that to third parties.

In embedded scenarios, **`Embassy`** is the most mainstream modern embedded async framework. It provides an **executor** to schedule tasks, as well as a **hardware abstraction layer** to operate peripherals.

Note that `Embassy` **is not suitable for scenarios requiring strict hard real-time guarantees**: it uses soft real-time cooperative scheduling and requires tasks to voluntarily yield the CPU. This means the response speed of concurrent scheduling is limited, and real-time behavior depends entirely on the developer's code design — poor code can be severely blocked even under an async framework.

Therefore, besides `Embassy`, Rust has another **hard real-time async framework, `RTIC`**.

- Mechanism: define the priorities of interrupt handlers (ISRs) in code. When a high-priority peripheral interrupt fires, the CPU hardware immediately interrupts the task currently executing (**even if it is in the middle of an .await point**) to execute the interrupt service function.

- Real-time behavior: the response latency depends on the hardware characteristics of the CPU's interrupt response (typically nanoseconds to microseconds), and **does not depend on whether a task voluntarily yields**. This guarantees that high-real-time tasks will absolutely never be blocked by other business logic.

In real projects, these two frameworks can be used together, which is a recommended and common practice in both the community and industry.

## Comparison with RTOS

| Dimension | Traditional RTOS (e.g. FreeRTOS) | Rust async (e.g. Embassy) |
| --------- | --------------------------------- | ----------------------------------- |
| **Scheduling** | **Preemptive**: a high-priority task can interrupt a low-priority task at any time. | **Cooperative**: tasks must voluntarily yield the CPU. |
| **Real-time behavior** | **Hard real-time**, predictable, with clear priorities. | **Soft real-time**, hard to predict, dependent on code design. |
| **Context switch** | **Scheduled by the kernel**: relatively costly, requiring the full task context to be saved and restored. | **Occurs at `.await` points**: extremely low overhead, only the state machine needs to be switched. |
| **Memory usage** | **Higher**: each task's stack occupies RAM. | **Extremely low**: a single task may need only **a few dozen bytes**, with all tasks sharing one stack. |
