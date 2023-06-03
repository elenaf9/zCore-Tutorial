# User-state synchronous mutual exclusion: Futex objects

## Introduction to the Futex mechanism

> Futex is the only underlying facility for user-state synchronous mutual exclusion in modern OS
>
> Why it's fast: use atomic variables in shared memory to avoid going into the kernel

Futexes are kernel primitives that are used together with user-space atomic operations for efficient synchronization primitives (e.g. Mutexes, Condition Variables, etc.) that require system calls only in contended cases. Usually they are implemented in the standard library.

## Implementing the base metaphrases: wait and wake

## Implement wait and wake functions and do unit tests

## Implement advanced operations

> implement complex APIs defined in Zircon
