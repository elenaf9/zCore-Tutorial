# zCore's user state runtime support

The development of the libos version of zCore (uzCore for short) was carried out in parallel with the bare-metal version of zCore (bzCore for short), and both versions of zCore share all code except for the HAL layer. In order to support the normal operation of uzCore, zCore has made some modifications to the original Zircon/Linux design in terms of address space partitioning, and the Fuchsia source code has been simply modified and recompiled for this purpose; in addition, the Hardware Associated Layer (HAL) required by uzCore will be fully supported by the host OS, and a reasonable HAL layer In addition, the hardware related layer (HAL) required by uzCore will be fully supported by the host OS, a reasonable HAL layer interface division is also an important consideration to support uzCore.

## HAL layer interface design

The design of the HAL layer evolved during the development of bzCore and uzCore. During the development process, hardware implementation-related interfaces, such as page tables and physical memory allocation, are encapsulated and exposed to the upper kernel object layer. In the kernel-hal module, empty weakly linked implementations are given and the developers of bzCore or uzCore implement the corresponding interfaces and replace the pre-defined weakly linked empty functions by setting the function link names. Throughout the development process, the requirements for the HAL layer were continuously proposed and implemented, resulting in the first version of the HAL layer interface, which is designed to meet the functionality required by the existing kernel object implementation.

For the kernel object layer, the hardware environment is no longer the physical memory, CPU, MMU, etc. that can be seen in the real hardware environment, but a set of interfaces that HAL exposes to the upper layers. Zircon wraps the x86_64 and ARM64 hardware architectures, but does not provide a unified set of hardware APIs for the upper kernel objects to use directly. This allows development for two hardware architectures at the same time. In zCore's kernel object layer implementation, the underlying hardware interface can be completely disregarded, allowing a set of kernel object modules to run on both bzCore and uzCore, and later, if zCore further supports the RISC-V 64 architecture (which has been initially implemented), only a new HAL implementation is needed, without modifying the upper code. The following will list the current HAL layer of uzCore, i.e. the kernel-hal-unix interface.


### **HAL interface name Function description**

* Thread-related
    * hal_thread_spawn Thread::spawn creates a new thread and adds it to the schedule
    * hal_thread_set_tid Thread::set_tid sets the id of the current thread
    * hal_thread_get_tid Thread::get_tid Get the id of the current thread
* future
    * yield_now temporarily relinquishes the CPU and returns to the async runtime
    * sleep_until hibernate until timed arrival
    * YieldFuture Abandon the executed future
    * sleepFuture A future that sleeps and waits to be woken up
    * SerialFuture Future that gets characters via serial_read
* Context switch related
    * VectorRegs x86 related
    * hal_context_run context_run enters "user state" run
* User pointer related
    * UserPtr operations on user pointers: read/write/de-reference/access arrays/access strings
    * IoVec non-contiguous buffer set (Vec structure): read/write
* Page table related
    * hal_pt_currentPageTable::current Get current page table
    * hal_pt_newPageTable::new Create a new page table
    * hal_pt_map PageTable::map Maps a physical page frame to a virtual address
    * hal_pt_unmap PageTable::unmap unmaps a virtual address
    * hal_pt_protect PageTable::protect Modify the flags of the page table entry corresponding to vaddr
    * hal_pt_query PageTable::query Query the status of the page table entry corresponding to a virtual address
    * hal_pt_table_phys PageTable::table_phys Get the physical address of the root table of the corresponding page table
    * hal_pt_activate PageTable::activate Activate the current page table
    * PageTable::map_many Map multiple physical page frames to contiguous virtual memory space at the same time
    * PageTable::map_cont Map multiple contiguous physical page frames to virtual memory space at the same time
    * hal_pt_unmap_cont PageTable::unmap_cont unmap a range starting from a virtual address
    * MMUFlags Attribute bits for page table entries
* Physical page frame related
    * hal_frame_alloc PhysFrame::alloc Allocate a physical page frame
    * hal_frame_alloc_contiguous PhysFrame::alloc_contiguous_base Allocates a contiguous piece of physical memory
    * PhysFrame::addr returns the physical address of the physical page frame
    * PhysFrame::alloc_contiguous allocates a contiguous piece of physical memory
    * PhysFrame::zero_frame_addr returns the physical address of a zero page (a special page whose contents are always all zeros)
    * PhysFrame::drop drop trait reclaims the physical page frame
    * hal_pmem_read pmem_read Reads the contents of a particular physical page frame into a buffer
    * hal_pmem_write pmem_write writes the contents of the buffer to a specific physical page frame
    * hal_frame_copy frame_copy copies the contents of a physical page frame
    * hal_frame_zero frame_zero_in_range Zeroes a physical page frame
    * hal_frame_flush frame_flush flushes the data of the physical page frame from the Cache back to memory
* Basic I/O peripherals
    * hal_serial_read serial_read string input
    * hal_serial_write serial_write string output
    * hal_timer_now timer_now Get current time
    * hal_timer_set timer_set sets a clock and calls the callback function when the deadline is reached
    * hal_timer_set_next timer_set_next Set the next clock
    * hal_timer_tick timer_tick is the clock function that will be called when the clock interrupt is generated, triggering the callback of all the times that have been reached
* Interrupt handling
    * hal_irq_handle handle interrupt handling routines
    * hal_ioapic_set_handle set_ioapic_handle x86 related, set handler routine for advanced interrupt controller
    * hal_irq_add_handle add_handle Add interrupt handling routine for an interrupt
    * hal_ioapic_reset_handle reset_ioapic_handle Reset the level interrupt controller and set the handler routine
    * hal_irq_remove_handle remove_handle Remove the interrupt handler routine for an interrupt
    * hal_irq_allocate_block allocate_block Allocate a contiguous area to an interrupt
    * hal_irq_free_block free_block Free a contiguous area for an interrupt
    * hal_irq_overwrite_handler overwrite_handler Overwrite the interrupt handler routine for an interrupt
    * hal_irq_enable enable enable an interrupt
    * hal_irq_disable disable block an interrupt
    * hal_irq_maxinstr maxinstr x86 related, get maxinstr for IOAPIC????
    * hal_irq_configure configure configure a certain interrupt????
    * hal_irq_isvalid is_valid Query whether an interrupt is valid or not
* Hardware platform related
    * hal_vdso_constants vdso_constants Get platform-related constant parameters
        * struct VdsoConstants Platform-related constants:

max_num_cpus features dcache_line_size ticks_per_second ticks_to_mono_numerator ticks_to_mono_denominator physmem version_string_len  version_string

    * fetch_fault_vaddr fetch_fault_vaddr Get the address of the error ???? It seems that hal_* is missing
    * fetch_trap_num fetch_trap_num Get the interrupt number
    * hal_pc_firmware_tables pc_firmware_tables x86 related, get the physical address of `acpi_rsdp` and `smbios`
    * hal_acpi_table get_acpi_table get acpi table
    * hal_outpd outpd x86 related, write access to IO Port
    * hal_inpd inpd x86 related, read access to IO Port
    * hal_apic_local_id apic_local_id Gets the local (local) APIC ID
    * fill_random generates random numbers and writes them to the buffer

In the above "thread related" list, some of the interfaces designed for the HAL layer are listed, covering the thread scheduling aspect. For thread scheduling, the interface related to the Thread structure is mainly used for basic operations such as adding a thread to the schedule. In the zCore implementation, the thread scheduling interfaces are implemented using the interface given by naive-executor and the interface given by trapframe, both of which are Rust libraries that we have packaged for concurrent scheduling and context switching in a bare-metal environment. uzCore relies on Rust's user-state concurrent support for thread scheduling and the In uzCore, the thread scheduling interface relies on Rust's user-state concurrency support and the user-state context switching implemented by the uzCore developers.

For memory management, the HAL layer divides memory management into page table operations and physical page frame management, and designs the interface accordingly. In the zCore implementation, the allocation and recovery of physical page frames is implemented directly in the general control module zCore because it requires the design of a physical page frame allocator and the size of the allocable range is closely related to the memory probe when the kernel is first started. In uzCore, the page table operations are simulated by mmap, while the physical page frame operations are simulated directly using the user-state physical memory allocator.

In order to meet this requirement at the kernel object level, we design a zero-page interface for the HAL layer, which requires the HAL layer to reserve a physical page frame with all zeros for use by the upper layers. The upper layer is responsible for ensuring that the contents of the zero page are not modified.


## Modify VDSO

VDSO is a dynamic link library provided by the kernel and mapped read-only to the user state, providing a system call interface in the form of a function interface. The original VDSO will eventually use the syscall instruction to get from the user state to the kernel state. However, in the uzCore environment, both kernel and user programs run in the user state, so the syscall instruction needs to be changed to a function call, i.e., the sysall instruction needs to be changed to a call instruction. To do this, we modified the VDSO assembly code to replace the syscall with a call, and made it available to uzCore. During the initialization of the uzCore kernel, we fill in the target address of the call instruction to redirect to a specific function in the kernel that handles the syscall, thus achieving the effect of simulating a system call.





## Adjusting the address space range

In uzCore, mmap is used to emulate the page table, and all processes share a 64-bit address space. Therefore, the address space of the user process running on uzCore is not as large as required by Zircon from the point of view of address space scope. For this reason, we manually allocate the address space for each user process to 0x100_0000_0000, starting from 0x2_0000_0000. 0x0 to 0x2_0000_0000 is the address space of the uzCore kernel and is not used for mmap. Figure 3.3 shows the address space distribution of uzCore for several user processes at runtime.

Compatible with uzCore, zCore follows the same design for the address space division of user processes, but in a bare-metal environment, it is somewhat freed from the limitation of separating different user address spaces in different page tables. As shown in Figure 3.4, zCore maps the address space of the three user processes in different page tables, but in order to be compatible with uzCore operation, the part of the address space of each user process that is actually accessible to the user program is only 0x100_0000_0000 in size.

## LibOS source code analysis log

### LibOS support for zCore on riscv64

* LibOS unix mode entry is in linux-loader main.rs:main()

initialization including kernel_hal_unix, Host filesystem, where loading of elf applications is done in the same way as in zcore bare mode;

The focus should be on the processing of **kernel state and user state switching to each other** in kernel_hal_unix.

kernel_hal_unix initialization includes, among other things, the processing function for the SIGSEGV signal when building Segmentation Fault, which is triggered when the code tries to use the fs register;

* Why do we need to register this signal handling function?

According to wrj's description: Since macOS user programs cannot modify the fs register, when running the relevant instructions will access illegal memory addresses to trigger Segmentation Fault. therefore, the segment error signal handling function is implemented, and the user program instructions are dynamically modified in it, changing fs to gs

kernel_hal_unix also constructs the run_fncall() -> syscall_fn_return() needed to **enter the user state**;

while the user program needs to call syscall_fn_entry() to **return to the kernel state**;

Linux-x86_64 platform runs with the fs base register applied for switching between user and kernel states;

* How to set fsbase / gsbase via system calls under Linux and macOS respectively.

This conversion process calls into the trapframe library, which is implemented for x86_64 and aarch64, while riscv needs to be implemented manually;

* About fs registers

fs registers are generally used to address TLS, and each thread has its own fs base address;

fs registers are defined by glibc to hold tls information, and the structure tcbhead_t is used to describe tls;

The kernel stack pointer is stored in the TLS region of the kernel glibc before entering the user state.

A code conversion tool for runtime programs can be found at [https://github.com/DynamoRIO/dynamorio/issues/1568#issuecomment-239819506](https://github.com/DynamoRIO/ dynamorio/issues/1568#issuecomment-239819506?fileGuid=VMAPV7ERl7HbpNqg)

* **LibOS kernel state and user state switching**

Linux x86_64, the fs registers cannot be set by user state programs, but only by system calls;

The clone system call, for example, sets the fs register via arch_prctl; the struct pthread pointed to, in glibc, where the first structure is tcbhead_t

Calculating the tls structure offset:

After testing, x86_64 platform, int type: 4 sections, pointer type: 8 sections, unsigned long integer type: 8 sections;

riscv64 platform, int type: 4 sections, pointer type: 8 sections, unsigned long integer type: 8 sections;

When calculating tls offsets, note that in musl, the aarch64 and riscv64 architectures have #define TLS_ABOVE_TP, while x86_64 does not have this definition

* About Linux user mode (UML)

"No, UML works only on x86 and x86_64."

[https://sourceforge.net/p/user-mode-linux/mailman/message/32782012/](https://sourceforge.net/p/user-mode-linux/mailman/message/ 32782012/?fileGuid=VMAPV7ERl7HbpNqg)
