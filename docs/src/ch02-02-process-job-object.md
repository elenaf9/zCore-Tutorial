# Process management: Process and Job objects



> Introduction to the overall design of Process and Job
>
> Basic framework for implementing Process and Job objects, supporting tree structure
>

## Job
### Overview
A job is a group of processes, and possibly other (sub)jobs. Jobs are used to keep track of the privilege of performing kernel operations (i.e. various syscalls using various options), as well as to track and limit the consumption of basic resources (e.g. memory, CPU). Each process belongs to a job. Jobs can also be nested, and each job except the root job belongs to a (parent) job.
### Description

A job is an object that contains the following:

- A reference to the parent job
- A set of child jobs (the parent of each child job is both this job)
- A set of member processes
- A set of policies

An "application" consisting of multiple processes can be controlled by a job as a single entity based on a set of policies.

### Job Policy 

The [policy policy](https://fuchsia.dev/fuchsia-src/concepts/settings/policy/policy_concepts?hl=en) can dynamically modify various configurations (settings) of the system while the Kernel is running. Job policies are mainly concerned with job security and condition restrictions on resource usage.

#### Policy ActionsPolicyAction

The policy actions include

- Allow Allow conditions
- Deny Deny condition
- AllowException generates an exception through the debugt port and resumes execution after the exception is handled and the condition is run
- DenyException Generates an exception through the debugt port and resumes execution after the exception is handled
- Kill Kills the process.

#### Conditions when applying policies PolicyCondition

The conditions when applying policy include

- BadHandle: A process under this job is trying to issue a syscall with an invalid handle. in this case, `PolicyAction::Allow` and `PolicyAction::Deny` are equivalent: if the syscall returns, it will always return the error ZX_ERR_BAD_HANDLE. HANDLE.
- WrongObject: A process under this job is trying to issue a syscall with a handle that does not support the action.
- VmarWx: A process under this job is trying to map an address area that has write access.
- NewAny: A special condition representing all of the above ZX_NEW conditions, such as NEW_VMO, NEW_CHANNEL, NEW_EVENT, NEW_EVENTPAIR, NEW_PORT, NEW_SOCKET, NEW_FIFO and any future ZX_NEW policies. This will include all new kernel objects that do not require a parent object to create.
- NewVMO: A process under this job is trying to create a new vm object.
- NewChannel: A process under this job is trying to create a new channel.
- NewEvent: A process under this job is trying to create a new event.
- NewEventPair: A process under this job is trying to create a new event pair.
- NewPort: A process under this job is trying to create a new port.
- NewSocket: A process under this job is trying to create a new socket.
- NewFIFO: A process under this job is trying to create a new FIFO.
- NewTimer: A process under this job is trying to create a new timer.
- NewProcess: A process under this job is trying to create a new process.
- NewProfile: A process under this job is trying to create a new profile.
- AmbientMarkVMOExec: A process under this job is trying to use zx_vmo_replace_as_executable() with ZX_HANDLE_INVALID as the second argument instead of the valid ZX_RSRC_KIND_VMEX.

## ProcessProcess

A process is a running instance of a program in the traditional sense, containing a set of instructions and data that will be executed by one or more threads, and having a set of resources. In terms of concrete implementation, a process consists of the following:

- Handles: mostly handles to resource objects used by the process
- Virtual Memory Address Regions: the memory address space where the process is located
- Threads: the group of threads that the process contains



Processes are included in the scope of Job management. From the point of view of resource and permission constraints and lifecycle control, it is allowed to consider an application consisting of multiple processes as a single entity (i.e. a job).

### Life cycle (lifetime)
Processes have their own lifecycle, from the time they are created until they are forced to terminate or the program exits. A process can be created by calling `Process::create()` and started by calling `Process::start()` . The process stops executing when

- The last thread terminates or quits
- The process calls `Process::exit()`
- The parent job terminates the process
- The parent job is destroyed (destroied)

Note: `Process::start()` cannot be called twice. New threads cannot be added to a process that has already been started.
