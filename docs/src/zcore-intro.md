# zCore overall structure and design patterns

First of all, from [Design and Implementation of Rust Language Operating System,Wang Runji B.S. Thesis,2019](https://github.com/rcore-os/zCore/wiki/files/wrj-thesis.pdf) and [Design and Implementation of zCore OS Kernel,Pan Qinglin B.S. Thesis. 2020](https://github.com/rcore-os/zCore/wiki/files/pql-thesis.pdf), we can get a full picture of the design process from the design of rCore to the design of zCore.

## The overall structure of zCore

The overall structure/project design diagram of [zCore](https://github.com/rcore-os/zCore) is as follows:

! [img](zcore-intro/structure.svg)

The design of zCore has two main starting points:

- Encapsulation of kernel objects: encapsulating the kernel object code into a library to ensure reusability
- Hardware interface design: make the design of hardware and kernel objects relatively independent, and only provide a unified, abstract API interface upwards

The project is designed from top to bottom, with the top layer further away from the hardware and the bottom layer closer to the hardware.

The top layer of the zCore design is the upper operating system, such as zCore, rCore, Zircon LibOS, and Linux LibOS, and there is some common code for each version of the operating system in the project architecture. The part related to the zCore microkernel design implementation is mainly the blue line on the left side of the figure.

The second layer, the ELF Program Loader, consists of the zircon-loader and the linux-loader, which encapsulates the logic for initializing kernel objects, initializing some hardware, setting up the system call interface, running the first user-state program, etc., and forming a library function. zCore is implemented at the top level by calling the zCore enters the first user-state program execution by calling the initialization logic in the zircon-loader library.

The third layer, the Syscall Implementation layer, includes zircon-syscall and linux-syscall. This layer encapsulates all system call processing routines into a library of system calls for use by the operating system above.

The fourth layer implements Kernel Objects using the virtual hardware APIs provided by the hardware abstraction layer, and implements the specific processing routines needed for each system call interface in the third layer based on the various Kernel Objects implemented.

The fifth layer is the Hardware Abstraction Layer (HAL), which corresponds to the kernel-hal module. kernel-hal provides all the interfaces needed to operate the hardware upwards, thus making the hardware environment transparent to the upper operating system.

The sixth layer is a layer of encapsulation of the code for direct hardware manipulation, corresponding to the kernel-hal-bare and kernel-hal-unix modules. kernel-hal family of libraries is only responsible for the interface definition, i.e. translating the underlying hardware/host OS operations into a form that can be used by the upper OS. In this case, kernel-hal-bare is responsible for translating bare metal hardware functions, while kernel-hal-unix is responsible for translating system calls for Unix-like systems.

At the bottom is the underlying runtime environment, which includes Bare Metal (bare metal), Linux / macOS operating systems. bare metal can be thought of as hardware interfaces such as registers on the hardware architecture.

## zCore kernel components

The zCore kernel runtime component hierarchy is outlined as follows:

! [image-20200805123801306](zcore-intro/image-20200805123801306.png)

During zCore startup, various components such as physical page frame allocator, heap allocator, thread scheduler, etc. are initialized. The zircon-loader is entrusted with the initialization of the kernel objects, and then the user-state boot process begins. Whenever the user state triggers a system call into the kernel state, the system call processing routine will process the service request through the implemented kernel object; the underlying operations required for the internal implementation of the corresponding kernel object are provided by the kernel components through the HAL layer interface.

Among them, VDSO (Virtual dynamic shared object) is a so file mapped to user space that can execute some simple system calls without getting caught in the kernel. Executor is a Rust-based `async` mechanism in zCore for the concurrent scheduler.

The HAL interface layer is also designed to take advantage of Rust's ability to specify function linking procedures. That is, all virtual hardware interfaces that can be called by the zircon-object and zircon-syscall libraries are specified in kernel-hal as function APIs, but are internally unimplemented and set to a weakly referenced linking state. The concrete implementation of the hardware interfaces in the bare-metal environment is given in kernel-hal-bare, and the unimplemented interfaces of the same name in kernel-hal will be replaced/overwritten during the linking process when the zCore project is compiled, so that the HAL layer can be flexibly selected during compilation.
