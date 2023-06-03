# Zircon Task Management System

Threads represent multiple execution control streams (CPU registers, stack, etc.) in the address space owned by the containing process (Proess). Processes are Jobs, which define various resource constraints. Jobs are owned by their parent Jobs until the Root Job, which is created by the kernel at boot time and passed to [`userboot` (the first user process to start execution)](https://fuchsia.dev/docs/concepts/booting/ userboot).

If there is no job handle (Job Handle), a thread in the process cannot create another process or another job.

[Program Load](https://fuchsia.dev/docs/concepts/booting/program_loading) is provided by userspace tools and protocols above the kernel level.

Some related system calls:

 [`zx_process_create()`](https://fuchsia.dev/docs/reference/syscalls/process_create), [`zx_process_start()`](https://fuchsia.dev/) docs/reference/syscalls/process_start), [`zx_thread_create()`](https://fuchsia.dev/docs/reference/syscalls/thread_create), [`zx_ thread_start()`](https://fuchsia.dev/docs/reference/syscalls/thread_start)
 