# Thread Management: Thread Object


A thread object is a structure that represents the execution context of a time-sharing CPU. The thread object is associated with a specific process object that provides the necessary memory and handles to other objects for the I/O and computation involved in the thread object's execution.




## Lifecycle


Threads are created by calling Thread::create(), but only start executing when Thread::create() or Process::start() is called. The arguments to both of these system calls are the entry points for the initial routines to be executed.


The thread passed to Process::start() should be the first thread to start execution on a process.




Any of the following can cause a thread to terminate its execution:


- By calling `CurrentThread::exit()`
- When the parent process terminates
- By calling `Task::kill()`
- After generating an exception where there is no handler or where the handler decides to terminate the thread.


Returning from the entry routine does not terminate execution. The last action at the entry point should be a call to CurrentThread::exit().






Closing the last handle of a thread does not terminate execution. To forcibly kill a thread that does not have a handle available, you can use KernelObject::get_child() to get a handle to the thread. However, this method is highly undesirable. Killing an executing thread may leave the process in a corrupted state.


Local threads are always detached (*detached*). That is, a join() operation is not needed to do a clean termination. However, some systems running on top of the kernel, such as C11 or POSIX, may require threads to be joined (be joined).






## Signals


Threads provide the following signals:


- THREAD_TERMINATED
- THREAD_SUSPENDED
- THREAD_RUNNING


THREAD_RUNNING is set when a thread starts execution. When it is suspended, THREAD_RUNNING is cancelled and THREAD_SUSPENDED is set. When the thread resumes, THREAD_SUSPENDED is cancelled and THREAD_RUNNING is set. When the thread terminates, THREAD_RUNNING and THREAD_SUSPENDED are both set, and THREAD_TERMINATED is also set.


Note that the signals enter the state held by the KernelObject::wait_signal() family of functions after the "or" operation, so when they return, you may see any combination of the requested signals.






## ThreadState


> State transfer: create -> run -> pause -> exit, preferably with a diagram of the state machine
>
> Implement ThreadState, preferably with a unit test to verify the transfer process


```rust
pub enum ThreadState {
    New, \\\ the thread has been created, but not yet running
    Running, \\\ the thread is running the user code normally
    Suspended, \\\ suspended due to zx_task_suspend()
    Blocked, \\\blocking in a system call or handling an exception
    Dying, \\\the thread is in the process of being terminated, but has not yet stopped running
    Dead, \\\the thread has stopped running
    BlockedException, \\\the thread is blocked in an exception
    BlockedSleeping, \\\the thread is blocked in zx_nanosleep()
    BlockedFutex, \\\the thread is blocked in zx_futex_wait()
    BlockedPort, \\\ the thread is blocked in zx_port_wait()
    BlockedChannel, \\\ the thread is blocked in zx_channel_call()
    BlockedWaitOne, \\\the thread is blocked in zx_object_wait_one() 
    BlockedWaitMany, \\\the thread is blocked in zx_object_wait_many()
    BlockedInterrupt, \\\the thread is blocked in zx_interrupt_wait()
    BlockedPager, \\\ blocked by Pager (currently not used????)
}
ðŸ™' ðŸ™'






## Thread register context


> Define ThreadState, implement read_state, write_state






## Async runtime and HAL hardware abstraction layer


> Brief introduction to async-std asynchronous mechanism
>
> Introduce HAL implementation method: weak link
>
> Implementation of hal_thread_spawn


## Thread start


> Connect HAL to Thread::start and write unit tests to verify that multiple threads can be started
