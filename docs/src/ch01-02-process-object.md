#### Object Manager: Process objects

## Handles - a bridge to manipulate kernel objects

In 1.1 we implemented a core kernel object in the Rust language. In this subsection we will step through the other two of the three important concepts related to kernel objects: **Handle and Rights**.

A handle is a kernel structure that allows a user program to refer to a kernel object reference, which can be thought of as a session or connection to a specific kernel object.

Typically, multiple processes access the same object at the same time through different handles. Objects may have multiple handles (in one or more processes) referencing them. However, a single handle can only be bound to a single process or bound to the kernel.

### Defining handles

Define a submodule under the object module:

``rust,noplaypen
// src/object/mod.rs
mod handle.

pub use self::handle::*.
```

Define handle:

```rust,noplaypen
// src/object/handle.rs
use super::{KernelObject, Rights}.
use alloc::sync::Arc.

/// Kernel object handle
#[derive(Clone)]
pub struct Handle {
    pub object: Arc<dyn KernelObject>.
    pub rights: Rights.
}
```

A Handle contains two fields, object and right. object is the kernel object that implements the `KernelObject` Trait, and Rights is the rights of the handle, which we will mention below.

Arc<T> is a reference count type that can be used on multiple threads. This count increases as `Arc<T>` is created or copied, and decreases when `Arc<T>` is recycled at the end of its life cycle. After this count becomes zero, the count variable itself and the referenced variables are reclaimed from the heap.

Why do we use Arc smart pointers here?

The vast majority of kernel object destructions occur when the handle count is zero, when the last Handle pointing to the kernel object is closed and the object dies, or enters a final state that cannot be undone. Obviously, this is a natural fit with Arc<T>.

## Permissions to control handles - Rights

One of the fields in the Handle above is rights, which is the handle's permissions. As the name implies, rights specify what operations the handle can perform on the referenced object.

When different permissions are bound to the same object, different handles are also formed.

### Defining Permissions

Define a submodule under the object module:

```rust,noplaypen
// src/object/mod.rs
mod rights.

pub use self::rights::*.
```

The rights are a number in u32


```rust,noplaypen
// src/object/rights.rs
use bitflags::bitflags.

bitflags!
    /// handle rights
    pub struct Rights: u32 {
        const DUPLICATE = 1 << 0.
        const TRANSFER = 1 << 1.
        const READ = 1 << 2.
        const WRITE = 1 << 3.
        const EXECUTE = 1 << 4.
		...
    }

```

[**bitflags**](https://docs.rs/bitflags/1.2.1/bitflags/) is a crate commonly used for bitflags in Rust. It provides a `bitflags!` macro, and with the help of the `bitflags!` macro we wrap the rights of a `u32` into a `Rights` structure, as shown in the snippet above. Note that before we can use it, we need to introduce the dependencies of the crate:

```rust,noplaypen
# Cargo.toml

[dependencies]
bitflags = "1.2"
```

After defining the permissions, we go back to the implementation of the handle-related methods.

The first and easiest part is to create a handle. Obviously we need to provide two parameters, the kernel object the handle is associated with and the handle's permissions.

```rust,noplaypen
impl Handle {
    /// Create a new handle
    pub fn new(object: Arc<dyn KernelObject>, rights: Rights) -> Self {
        Handle { object, rights }
    }
}
```

### Testing

Alright, let's test it!

```rust,noplaypen
#[cfg(test)
mod tests {
    use super::*.
    use crate::object::DummyObject.
    
    #[test]
    fn new_obj_handle() {
        let obj = DummyObject::new().
        let handle1 = Handle::new(obj.clone(), Rights::BASIC).
    }
}
```

## Handle storage carrier - Process

After implementing the handle, we start to consider, where is the handle stored?

From the previous explanation, it is clear that Process owns the kernel object handle, that is, the handle is stored in Process, so let's implement a Process first: ### Implement an empty process object

### Implement the empty process object

```rust,noplaypen
// src/task/process.rs
//// Process object
pub struct Process {
    base: KObjectBase.
    inner: Mutex<ProcessInner>.
}
// The role of the macro: add
impl_kobject!(Process).

struct ProcessInner {
    handles: BTreeMap<HandleValue, Handle>.
}

pub type HandleValue = u32.
```

handles uses BTreeMap to store the key is HandleValue and the value is the handle. HandleValue is actually an alias for the u32 type.

Wrapping the internal object ProcessInner in a spinlock Mutex guarantees mutually exclusive access, because Mutex will help us handle concurrency, as explained in detail in section 1.1.

Next we implement the method that creates a Process:

``rust,noplaypen
process {
    /// Create a new process object
    pub fn new() -> Arc<Self> {
        Arc::new(Process {
            Base: KObjectBase::default(), the
            internal: Mutex::new(ProcessInner {
                Process: BTreeMap::default(), }
            }).
        })
    }
}
```

#### unit test

We have implemented the method to create a Process, and we write a unit test for the following:

``rust,noplaypen
#[test]
    fn new_proc() {
        let proc = Process::new();
        assert_eq! (proc.type_name(), "Process");
        assert_eq! (proc.name(), "");
        proc.set_name("proc1");
        assert_eq! (proc.name(), "proc1");
        assert_eq!
            format! ("{:?}" , proc),
            format!( "Process({}, \"proc1\")", proc.id()
        ).

        let obj: Arc<dyn KernelObject> = proc;
        assert_eq! (obj.type_name(), "Process");
        assert_eq! (obj.name(), "proc1");
        obj.set_name("proc2");
        assert_eq! (obj.name(), "proc2");
        assert_eq!
            format! ("{:?}" , obj),
            format!( "Process({}, \"proc2\")", obj.id()
        ).
    
```

### Process-related methods

#### Insert handle

Adds a new handle to the Process, the return value is a handleValue, which is u32:.

``rust,noplaypen
pub fn add_handle(&self, handle: Handle) -> HandleValue {

    let mut inner = self.inner.lock();
    let value = (0 as HandleValue...)
    	.find(|idx| !inner.handle.contains_key(idx))
    	.unwrap();
    // insert BTreeMap
    inner.handle.insert(value, handle);
    value
 }
```

#### Remove Handle

Removes a handle from a process:

``rust,noplaypen
pub fn remove_handle(&self, handle_value: HandleValue) {
	self.inner.lock().handle.remove(&handle_value);
}
```

#### Find kernel objects based on handles

```rust,noplaypen
// src/task/process.rs
impl Process {
    /// Find kernel objects based on handle values and check permissions
    pub fn get_object_with_rights<T: KernelObject>(
        &self.
        handle_value: HandleValue.
        desired_rights: Rights.
    ) -> ZxResult<Arc<T>> {
        let handle = self
            .inner
            .lock()
            .handles
            .get(&handle_value)
            .ok_or(ZxError::BAD_HANDLE)?
            .clone().
        // check type before rights
        let object = handle
            .object
            .downcast_arc::<T>()
            .map_err(|_| ZxError::WRONG_TYPE)? .
        if !handle.rights.contains(desired_rights) {
            return Err(ZxError::ACCESS_DENIED).
        }
        Ok(object)
    }
}
```

#### ZxResult

ZxResult is the i32 value representing the state of Zircon, with the value space divided as follows:

- 0:ok
- Negative values: defined by the system (i.e. this file)
- Positive values: are reserved for protocol-specific error values and are never defined by the system.

```rust,noplaypen
pub type ZxResult<T> = Result<T, ZxError>.

#[allow(non_camel_case_types, dead_code)]
#[repr(i32)]
#[derive(Debug, Clone, Copy)]
pub enum ZxError {
    OK = 0.
   	...
   	
    /// A specific handle value that does not point to a handle
    BAD_HANDLE = -11.
    
    /// The body of the operation is of the wrong type for the execution of this operation
    /// For example: try to execute message_read on thread handle.
    WRONG_TYPE = -12.
    
    // Permission check error
    // The caller does not have permission to execute the operation
    ACCESS_DENIED = -30.
}
```

ZxResult<T> is equivalent to Result<T, ZxError>, which is equivalent to defining our own kind of error.

### Unit testing

So far, we have implemented the most basic methods of Process, so let's run a unit test:

```rust,noplaypen
fn proc_handle() {
        let proc = Process::new().
        let handle = Handle::new(proc.clone(), Rights::DEFAULT_PROCESS).
        let handle_value = proc.add_handle(handle).

        let object1: Arc<Process> = proc
            .get_object_with_rights(handle_value, Rights::DEFAULT_PROCESS)
            .expect("failed to get object").
        assert!(Arc::ptr_eq(&object1, &proc)).

        proc.remove_handle(handle_value).
    }
```

## Summary

In this section we have implemented two important concepts of kernel objects, Handle and Rights, as well as Process, the carrier of handle storage, and implemented the basic methods of Process, which will be the basis for our continued exploration of zCore.

In the next section, we will introduce Channel, the transporter of kernel objects.
