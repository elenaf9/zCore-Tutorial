# User programs

zCore is designed in a microkernel style. One of the complexities of microkernel design is "how to boot the initial user space process". Usually this is achieved by having the kernel implement a minimal version of file system reads and program loads.
