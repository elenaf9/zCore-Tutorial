# Context switching

> This section describes the magic implementation of [fncall.rs] in trapframe-rs

[fncall.rs]: https://github.com/rcore-os/trapframe-rs/blob/master/src/arch/x86_64/fncall.rs

# # Save and restore general purpose registers

> Define the UserContext structure

```rust
pub struct UserContext {
    pub general: GeneralRegs,
    pub trap_num: usize,
    pub error_code: usize,
}
```
```rust
pub struct GeneralRegs {
    pub rax: usize,
    pub rbx: usize,
    pub rcx: usize,
    pub rdx: usize,
    pub rsi: usize,
    pub rdi: usize,
    pub rbp: usize,
    pub rsp: usize,
    pub r8: usize,
    pub r9: usize,
    pub r10: usize,
    pub r11: usize,
    pub r12: usize,
    pub r13: usize,
    pub r14: usize,
    pub r15: usize,
    pub rip: usize,
    pub rflags: usize,
    pub fsbase: usize,
    pub gsbase: usize,
}
```
`Usercontext` holds the context of user execution, including the address of the first instruction of the program after jumping to the user state, or the address of the first instruction of the rip pointing to the user process if the program first enters the user state from the kernel state for execution.
> save the callee-saved register to the stack, restore the UserContext register, and enter the user state, and vice versa
```rust
syscall_fn_return.
    # save callee-saved registers
    push r15
    push r14
    push r13
    push r12
    push rbp
    push rbx

    push rdi
    SAVE_KERNEL_STACK
    mov rsp, rdi

    POP_USER_FSBASE

    # pop trap frame (struct GeneralRegs)
    pop rax
    pop rbx
    pop rcx
    pop rdx
    pop rsi
    pop rdi
    pop rbp
    pop r8 # skip rsp
    pop r8
    pop r9
    pop r10
    pop r11
    pop r12
    pop r13
    pop r14
    pop r15
    pop r11 # r11 = rip. fixme: don't overwrite r11!
    popfq # pop rflags
    mov rsp, [rsp - 8*11] # restore rsp
    jmp r11 # restore rip
```
The popped registers correspond exactly to the structure of GeneralRegs. By calling the `syscall_fn_return` function in the unsafe code block of rust and passing a pointer to the `Usercontext` structure to rdi, you can create a runtime environment for the program to enter the user state.


## Finding the kernel context: thread-local storage and FS registers

> How to switch back to the kernel stack without destroying the user registers at the moment the user program jumps back to the kernel code?
>
> Before entering the user state, the kernel stack pointer is stored in the TLS area of the kernel glibc. To do this we need to look at the glibc source code and find a free location.
>
> How to set fsbase / gsbase via system calls on Linux and macOS respectively

## Testing

> Write unit tests to verify the above procedure
```rust
#[cfg(test)]
mod tests {
    use crate::*;

    #[cfg(target_os = "macos")]
    global_asm!(".set _dump_registers, dump_registers");

    // Mock user program to dump registers at stack.
    global_asm!(
        r#"
dump_registers:
    push r15
    push r14
    push r13
    push r12
    push r11
    push r10
    push r9
    push r8
    push rsp
    push rbp
    push rdi
    push rsi
    push rdx
    push rcx
    push rbx
    push rax

    add rax, 10
    add rbx, 10
    add rcx, 10
    add rdx, 10
    add rsi, 10
    add rdi, 10
    add rbp, 10
    add r8, 10
    add r9, 10
    add r10, 10
    add r11, 10
    add r12, 10
    add r13, 10
    add r14, 10
    add r15, 10

    call syscall_fn_entry
"#
    );

    #[test]
    fn run_fncall() {
        extern "sysv64" {
            fn dump_registers();
        }
        let mut stack = [0u8; 0x1000];
        let mut cx = UserContext {
            general: GeneralRegs {
                rax: 0,
                rbx: 1,
                rcx: 2,
                rdx: 3,
                rsi: 4,
                rdi: 5,
                rbp: 6,
                rsp: stack.as_mut_ptr() as usize + 0x1000,
                r8: 8,
                r9: 9,
                r10: 10,
                r11: 11,
                r12: 12,
                r13: 13,
                r14: 14,
                r15: 15,
                rip: dump_registers as usize,
                rflags: 0,
                fsbase: 0, // don't set to non-zero garbage value
                gsbase: 0,
            },
            trap_num: 0,
            error_code: 0,
        };
        cx.run_fncall();
        // check restored registers
        let general = unsafe { *(cx.general.rsp as *const GeneralRegs) };
        assert_eq!(
            general,
            GeneralRegs {
                rax: 0,
                rbx: 1,
                rcx: 2,
                rdx: 3,
                rsi: 4,
                rdi: 5,
                rbp: 6,
                // skip rsp
                r8: 8,
                r9: 9,
                r10: 10,
                // skip r11
                r12: 12,
                r13: 13,
                r14: 14,
                r15: 15,
                ..general
            }
        );
        // check saved registers
        assert_eq!(
            cx.general,
            GeneralRegs {
                rax: 10,
                rbx: 11,
                rcx: 12,
                rdx: 13,
                rsi: 14,
                rdi: 15,
                rbp: 16,
                // skip rsp
                r8: 18,
                r9: 19,
                r10: 20,
                // skip r11
                r12: 22,
                r13: 23,
                r14: 24,
                r15: 25,
                ..cx.general
            }
        );
        assert_eq!(cx.trap_num, 0x100);
        assert_eq!(cx.error_code, 0);
    }
}
```
## Trouble with macOS: Dynamic Binary Modification

> Since macOS user programs cannot modify the fs register, running the relevant instruction will access an illegal memory address and trigger a segment error.
> 
> We need to implement a segment error signal handling function and dynamically modify the user program instruction in it to change fs to gs.
