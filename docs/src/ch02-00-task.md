# Task management

In this chapter we will implement the first type of kernel object: Tasks.

Task objects include: Threads `Thread`, Processes `Process`, Jobs `Jobs`. and some auxiliary objects, such as `SuspendToken`, which is responsible for suspending the execution of a task, and `Exception`, which is responsible for handling exceptions.

In order to be able to realistically represent the behavior of thread objects, we use **user-state concurrent threads** in the Rust async runtime [`async_std`] to emulate **kernel threads**.
This allows the correctness of the implementation to be verified in a user-state unit test.
Considering that the OS will run in a bare metal environment in the future and will have different implementations of kernel threads, we create a special **Hardware Abstraction Layer (HAL)** to shield the underlying platform differences and provide a unified interface to the top.
This HAL interface will be expanded as needed in the future.

In this chapter, we will only implement the minimum subset of functionality necessary to run a program, leaving the rest to be implemented on demand once the user program is up and running.

[`async_std`]: https://docs.rs/async-std/1.6.2/async_std/index.html
