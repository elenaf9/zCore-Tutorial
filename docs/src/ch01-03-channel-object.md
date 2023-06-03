# Object transporter: Channel objects

## Overview

A Channel is a bidirectional message transfer consisting of a certain number of bytes of data and a certain number of handles.

## Kernel Objects for IPC

The main kernel objects used for IPC in Zircon are Channel, Socket and FIFO, the first two of which are described here.

> **Inter-Process Communication** (**IPC**, *Inter-Process Communication*), refers to some technique or method for transferring data or signals between at least two processes or threads. A process is the smallest unit of a computer system that allocates resources (a process is the smallest unit that allocates resources, while a thread is the smallest unit of scheduling, and threads share process resources). Each process has its own part of independent system resources that are isolated from each other. Inter-process communication is needed to enable different processes to access each other's resources and to coordinate their work. As a typical example, two applications that use inter-process communication can be classified as client and server, with the client process requesting data and the server replying to the client's data request. There are applications that are both servers and clients themselves, which can be seen from time to time in distributed computing. These processes can run on the same computer or on different computers connected by a network.

Both `Socket` and `Channel` are bidirectional and dual-ended IPC-related `Object`s. Creating a `Socket` or `Channel` will return two different `Handle`s that point to each end of the `Socket` or `Channel`. The difference with a channel is that a socket can only transfer data (and not move handles), while a channel can pass handles.

- A `Socket` is a stream-oriented object that can read or write data in units of one or more bytes.
- `Channel` is a packet-oriented object and limits the size of messages to a maximum of 64K (possibly smaller if there are changes) and a maximum of 1024 `Handles` mounted to the same message (again possibly smaller if there are changes).

When `Handle` is written to `Channel`, these `Handles` will be removed in the `Process` on the sending side. When the message carrying the `Handle` is read from the `Channel`, the `Handle` is also added to the receiving `Process`. Between these two points in time, `Handle`s will exist on both ends (to ensure that the `Object`s they point to continue to exist and are not destroyed), unless one end of the `Channel` write direction is closed, in which case the messages being sent to that end will be discarded, and any handles they contain will be closed.

## Channel

Channel is the only IPC that can pass a handle, the others can only pass messages. The channel has two endpoints `endpoints`, for code implementation ** the channel is virtual, we actually describe a channel in terms of two endpoints of the channel**. Each of the two endpoints has to maintain a message queue. Writing a message at one endpoint actually writes the message to the end of the message queue at **the other endpoint**; reading a message at one endpoint actually reads a message from the head of the message queue at **the current endpoint**.

Messages usually contain ``data`` and ``handles``; here we encapsulate the message in a ``MessagePacket`` structure, which contains the above two fields:

```rust,noplaypen
#[derive(Default)]
pub struct MessagePacket {
    /// message packet carries the data data
    pub data: Vec<u8>.
    /// message packet carries the handle Handle
    pub handles: Vec<Handle>.
}
```

### Implement the empty Channel object

Create an `ipc` directory in the `src` directory and define a submodule `channel` under the `ipc` module:

```rust,noplaypen
// src/ipc/mod.rs
use super::*.

mod channel.
pub use self::channel::*.
```

Introduce ``crate'' in ``ipc.rs'':

```rust,noplaypen
// src/ipc/channel.rs

use {
    super::*.
    crate::error::*.
    crate::object::*.
    alloc::collections::VecDeque.
    alloc::sync::{Arc, Weak}.
    spin::Mutex.
}.
```

Add the `MessagePacket` structure mentioned above to this file.

Next we add the Channel structure:

```rust,noplaypen
// src/ipc/channel.rs
pub struct Channel {
    base: KObjectBase.
    peer: Weak<Channel>.
    recv_queue: Mutex<VecDeque<T>>.
}

type T = MessagePacket.
```

`peer` represents the other endpoint of the pipeline where the current endpoint is located. The structures at each end hold a `Weak` reference to the other, and the structures at each end will be referenced by other data structures in the kernel as kernel objects via `Arc` references, which we will mention when creating the Channel instance.

`recv_queue` represents the message queue maintained by the current endpoint, which uses `VecDeque` to hold the `MessagePacket` and can be popped at the head of the queue and pressed in at the end of the queue using methods such as `pop_front()` and `push_back`.

Automatically implement the ``KernelObject`` trait using macros, using the channel type name, and adding two functions.

```rust,noplaypen
impl_kobject!(Channel
    fn peer(&self) -> ZxResult<Arc<dyn KernelObject>> {
        let peer = self.peer.upgrade().ok_or(ZxError::PEER_CLOSED)? .
        Ok(peer)
    }
    fn related_koid(&self) -> KoID {
        self.peer.upgrade().map(|p| p.id()).unwrap_or(0)
    }
).
```

### Implementing the method to create a Channel

Let's implement the method to create a `Channel` as follows:

```rust,noplaypen
impl Channel {

    #[allow(unsafe_code)]
    pub fn create() -> (Arc<Self>, Arc<Self>) {
        let mut channel0 = Arc::new(Channel {
            base: KObjectBase::default().
            peer: Weak::default().
            recv_queue: Default::default().
        }).
        let channel1 = Arc::new(Channel {
            base: KObjectBase::default(),
            peer: Arc::downgrade(&channel0),
            recv_queue: Default::default(),
        });
        // no other reference of `channel0`
        unsafe {
            Arc::get_mut_unchecked(&mut channel0).peer = Arc::downgrade(&channel1);
        }
        (channel0, channel1)
}
```
       The return value of this method is an `Arc` reference to the two endpoint structures (Channels), which will be referenced as kernel objects by other data structures in the kernel. The two endpoints hold each other's `Weak` pointers, because one endpoint does not need to have a reference count of 0. It can be cleaned up as long as `strong_count` is 0, even if the other endpoint points to it.

> The rust language does not provide garbage collection (GC, Garbage Collection), but it does provide the simplest reference counting wrapper type `Rc`, which is also a common method for early GC, but reference counting cannot fix circular references. So how to fix this circular reference? The answer is the `Weak` pointer, which only increases the reference logic and does not share ownership, i.e. it does not increase the strong reference count. since the object pointed to by the `Weak` pointer may be destructured, it cannot be directly dereferenced, it has to be pattern matched and then upgraded.

Let's analyze the `unsafe` code block as follows:

```rust,noplaypen
unsafe {
            Arc::get_mut_unchecked(&mut channel0).peer = Arc::downgrade(&channel1).
        }
```

Since the structures at each end will be referenced separately by `Arc` and used by other data structures in the kernel as kernel objects. Therefore, while initializing both ends at the same time, an operation to obtain a mutable reference to the Arc pointer at one end will have to be performed, i.e. the `get_mut_unchecked` interface. This interface is very insecure when the reference count of the `Arc` pointer is not `1`, but in the current context, we use this interface to initialize the `IPC` object, and security is guaranteed.

### Unit tests

Let's write a unit test to verify the `create` method we wrote:

```rust,noplaypen
#[test]
    fn test_basics() {
        let (end0, end1) = Channel::create().
        assert!(Arc::ptr_eq(
            &end0.peer().unwrap().downcast_arc().unwrap().
            &end1
        )).
        assert_eq!(end0.related_koid(), end1.id()).

        drop(end1).
        assert_eq!(end0.peer().unwrap_err(), ZxError::PEER_CLOSED).
        assert_eq!(end0.related_koid(), 0).
    }
```

### Implementing data transfer

Data transfer in Channel can be understood as the transfer of `MessagePacket` between two endpoints, so who can read and write messages?

There is a handle associated with the channel endpoint, and the process holding that handle is considered the owner (owner). So it is the process (that holds the handle associated with the channel endpoint) that can read or write the message, or send the channel endpoint to another process.

When `MessagePacket`s are written to the channel, they are removed from the sending process. When a `MessagePacket` is read from the channel, the handle to the `MessagePacket` is added to the receiving process.
#### read

Get the `recv_queue` of the current endpoint, read a message from the queue head, return `Ok` if the message can be read, otherwise return an error message.

```rust,noplaypen
pub fn read(&self) -> ZxResult<T> {
        let mut recv_queue = self.recv_queue.lock().
        if let Some(_msg) = recv_queue.front() {
            let msg = recv_queue.pop_front().unwrap().
            return Ok(msg).
        }
        if self.peer_closed() {
            Err(ZxError::PEER_CLOSED)
        } else {
            Err(ZxError::SHOULD_WAIT)
        }
    }
```

#### write

First get the `Weak` pointer of the other endpoint corresponding to the current endpoint and upgrade it to an `Arc` pointer via the `upgrade` interface to get the corresponding structure object. Push a `MessagePacket` at the end of its `recv_queue` queue.

```rust,noplaypen
pub fn write(&self, msg: T) -> ZxResult {
        let peer = self.peer.upgrade().ok_or(ZxError::PEER_CLOSED)? .
        peer.push_general(msg).
        Ok(())
    }
fn push_general(&self, msg: T) {
        let mut send_queue = self.recv_queue.lock().
        send_queue.push_back(msg).
    }
```

### Unit test

Here we write a unit test to verify the ``read`` and ``write`` methods we wrote above:

```rust,noplaypen
#[test]
    fn read_write() {
        let (channel0, channel1) = Channel::create().
        // write a message to each other
        channel0
            .write(MessagePacket {
                data: Vec::from("hello 1").
                handles: Vec::new().
            })
            .unwrap().

        channel1
            .write(MessagePacket {
                data: Vec::from("hello 0").
                handles: Vec::new().
            })
            .unwrap().

		// read message should succeed
        let recv_msg = channel1.read().unwrap().
        assert_eq!(recv_msg.data.as_slice(), b "hello 1").
        assert!(recv_msg.handles.is_empty()).

        let recv_msg = channel0.read().unwrap().
        assert_eq!(recv_msg.data.as_slice(), b "hello 0").
        assert!(recv_msg.handles.is_empty()).

        // read more message should fail.
        assert_eq!(channel0.read().err(), Some(ZxError::SHOULD_WAIT)).
        assert_eq!(channel1.read().err(), Some(ZxError::SHOULD_WAIT)).
    }
```

## Summary

In this section we implemented Channel, the only object transporter that can pass handles. We first learned about the main IPC kernel objects in Zircon, and then introduced the details of how to create and implement read and write functions for Channel.

In the next chapter, we will learn about `Zircon`s task management system and the process and thread management objects.
