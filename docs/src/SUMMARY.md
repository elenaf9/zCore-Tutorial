## Concise zCore Tutorials

[Concise zCore Tutorial](README.md)
[🚧 zCore overall architecture and design patterns](zcore-intro.md)
[🚧 Fuchsia OS and Zircon microkernel](fuchsia.md)

- [kernel-object](ch01-00-object.md)
    - [✅ Getting started with kernel objects](ch01-01-kernel-object.md)
    - [🚧 Object Manager: Process object](ch01-02-process-object.md)
    - [🚧 Object Transporter: Channel object](ch01-03-channel-object.md)

- [Task Management](ch02-00-task.md)
    - [🚧 Zircon Task Management System](ch02-01-zircon-task.md)
    - [🚧 Process Management: Process and Job Objects](ch02-02-process-job-object.md)
    - [🚧 Thread Management: Thread objects](ch02-03-thread-object.md)

- [Memory Management](ch03-00-memory.md)
    - [🚧 Zircon Memory Management Model](ch03-01-zircon-memory.md)
    - [🚧 Physical Memory: VMO Objects](ch03-02-vmo.md)
    - [🚧 Physical memory: VMO per-page allocation](ch03-03-vmo-paged.md)
    - [🚧 Virtual memory: VMAR objects](ch03-04-vmar.md)

- [User program](ch04-00-userspace.md)
    - [🚧 Zircon User Program](ch04-01-user-program.md)
    - [🚧 Context switching](ch04-02-context-switch.md)
    - [🚧 System calls](ch04-03-syscall.md)

- [signal-and-waiting](ch05-00-signal-and-waiting.md)
    - [🚧 Waiting for signals from kernel objects](ch05-01-wait-signal.md)
    - [🚧 Waiting for multiple signals at the same time: Port object](ch05-02-port-object.md)
    - [🚧 Implementing more: EventPair, Timer objects](ch05-03-more-signal-objects.md)
    - [🚧 User State Synchronization Mutex: Futex Objects](ch05-04-futex-object.md)

- [Hardware abstraction layer](ch06-00-hal.md)
    - [✅ UNIX hardware abstraction layer](ch06-01-zcore-hal-unix.md)
