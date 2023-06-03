# Kernel Objects

Zircon is a kernel object-based system. The functionality of the system is divided into several groups of kernel objects.

To start things off, this chapter first constructs a kernel object framework that serves as the basis for later implementations.
Then we implement the first kernel object, `Process`, which is the container for all objects and the entry point for future object manipulation.
Finally, we implement a slightly more complex but extremely important object, `Channel`, which is the infrastructure for inter-process communication (IPC) and is the only conduit for transferring objects.
