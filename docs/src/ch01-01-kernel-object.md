# Introduction to kernel objects

## Introduction to kernel objects

Before we can write our code, we need to do research and study first to get a comprehensive and systematic understanding of the target objects.
The best way to understand the design of a project is to read the official manuals and documentation provided.

Let's start by reading the official Fuchsia documentation: [Kernel Objects]. This link is to the community-translated Chinese version, which is a bit old. If the reader has scientific access to the Internet, we recommend reading the [official English version] directly.

[Kernel objects]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/objects.md
[Official English version]: https://fuchsia.dev/fuchsia-src/reference/kernel_objects/objects

By reading the documentation, we learn about three important concepts related to kernel objects: **Object, Handle, and Rights**. Their roles and relationships in the Zircon kernel are shown in the following diagram:

! [](img/ch01-01-kernel-object.png)

These three important concepts are defined as follows:

- **Object**: An object that has properties and behaviors. Objects can be associated with each other in various ways. From simple **integers** to complex **operating system processes**, they can be considered as objects, which can represent not only concrete things, but also abstract rules, plans or events.
- **Handle (Handle)**: a symbol that identifies an object, which can also be seen as a variable that points to an object (also known as an identifier, reference, ID, etc.).
- **Permissions (Rights)**: are the operations that the object's visitor is allowed to perform on the object, i.e., the object's access rights. When an object visitor opens a handle to an object, that handle has some combination of access rights to its object.

The relationship between Zircon and objects, handles, and permissions can be expressed simply as follows

* Zircon is an object-based kernel, where kernel resources are abstractly encapsulated in different **objects**.
* User programs interact with the kernel through **handles**. A handle is a reference to an object with specific **permissions** attached to it.
* Objects manage their life cycle through **reference counting**. For most objects, when the last handle pointing to it is closed, the object is then destroyed, or [enters an irretrievable final state](https://fuchsia.dev/fuchsia-src/concepts/kernel/concepts#handles_and_rights ).

Also in the documentation for kernel objects, there is a list of some [common objects]. Click on the link to see the [specific description] of this object, and at the bottom of the page there is a list of [all system calls] related to this object.
Looking further into the [API Definition] of the system call, and its [Behavior Description], we can go into more detail about how the user program operates the kernel object:

[common object]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/objects.md#应用程序可用的内核对象
[specific description]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/objects/channel.md
[All system calls]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/objects/channel.md#系统调用
[API Definition]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/channel_read.md#概要
[Description of behavior]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/channel_read.md#描述

* Create: A system call exists for each type of kernel object to create it, e.g. [`zx_channel_create`].
The object is usually created by passing a parameter option `options`, and if the creation is successful, the kernel writes a new handle to the user-specified memory.

* Use: After obtaining a handle to an object, it can be manipulated by several system calls, such as [`zx_channel_write`].
The kernel first checks the handle if it is illegal or if the object type does not match the system call.
Next, the kernel checks whether the handle's permission meets the operation requirements. For example, the `write` operation generally requires the handle to have the `WRITE` permission, and will continue to report an error if the permission is not met.

* Close: [`zx_handle_close`] is called when the user process is no longer using the object to close the handle. When the user process exits, any handles that are still open will also be closed automatically.

[`zx_channel_create`]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/channel_create.md
[`zx_channel_write`]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/channel_write.md
[`zx_handle_close`]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/handle_close.md

We also found that there is a class of Object system calls that are available for all kernel objects.
This means that all kernel objects have some common properties, such as ID, name, etc. Each kernel object will also have its own specific properties.

Some of the Object system calls are related to signals; Zircon has 32 **[Signals]** attached to each kernel object, which represent different types of events.
Unlike signals in traditional Unix systems, they cannot interrupt the user program asynchronously, but can only be actively blocked by the user program waiting for some signal on an object.
Signals are an important mechanism in the Zircon kernel, but we won't cover this part up front, so we'll leave the implementation to Chapter 5.

[Signals]: https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/signals.md

Above we have learned about the concepts and usage of Zircon kernel objects. In this section, we will implement the basic framework of kernel objects in Rust to facilitate the rapid implementation of various specific types of kernel objects.
From the perspective of a traditional object-oriented language, we are only implementing a base class. However, due to the limitations of the Rust language model, this matter requires some special skills.

## Building the project

First we need to install the Rust toolchain. On Linux or macOS systems, you can download and install rustup with a single command:

```sh
$ curl https://sh.rustup.rs -

You can refer to the [official documentation] for details on how to install it.

[Official Documentation]: https://kaisery.github.io/trpl-zh-cn/ch01-01-installation.html

Next we create a Rust library project with cargo:

```sh
$ cargo new --lib zcore
$ cd zcore
```

We will implement all the kernel objects in this crate, organizing the code as a library (lib) rather than an executable (bin), and later we will rely on unit tests to ensure the correctness of the code.

Since we will be using some unstable language features, we need to use the nightly version of the toolchain. Create a ``rust-toolchain`` file in the project root directory, specifying the version of the toolchain to use:

```sh
{{#include ... /... /code/ch01-01/rust-toolchain}}
```

This library is currently running on your Linux or macOS, but one day it will be a real OS running on bare metal.
To do this we need to remove the dependency on the standard library and make it a library that does not depend on the current OS functionality. Add the following declaration to the first line of ``lib.rs``:

``rust,noplaypen
// src/lib.rs
#! [no_std]
extern crate alloc.
```

Now we can try to run the self-contained unit tests, and the compiler may automatically download and install the toolchain at

```sh
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.52s
     Running target/debug/deps/zcore-dc6d43637bc5df7a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Implementing the KernelObject interface

All kernel objects have a set of properties and methods in common. We call the methods of an object the object's public **Interface**.
The same method may have different behavior in different types of objects, which we call **Polymorphism** in object-oriented languages.

Rust is a partially object-oriented language, and we usually use its traits to implement interfaces and polymorphism.

First create a ``KernelObject`` trait as a public interface to a kernel object:

```rust,noplaypen
use alloc::string::String.
// src/object/mod.rs
/// public interface for kernel objects
pub trait KernelObject: Send + Sync {
{{#include ... /... /code/ch01-01/src/object/mod.rs:object}}

{{#include ... /... /code/ch01-01/src/object/mod.rs:koid}}
```

Here [`Send + Sync`] is a precondition that binds all `KernelObject`s to be satisfied, i.e. it must be a **concurrent object**.
By concurrent object we mean that **it can be safely accessed by multiple threads on a shared basis**. In fact our kernel itself is a multi-threaded program sharing an address space, and each CPU core on a bare metal machine can be considered as a concurrent thread of execution.
Since a kernel object may be accessed by multiple threads at the same time, it must be a concurrent object.

[`Send + Sync`]: https://kaisery.github.io/trpl-zh-cn/ch16-04-extensible-concurrency-sync-and-send.html

## Implementing an empty object

Next we implement the simplest possible empty object, ``DummyObject``, and implement the ``KernelObject`` interface for it:

``rust,noplaypen
// src/object/object.rs
{{#include ... /... /code/ch01-01/src/object/object_v1.rs:dummy_def}}
```

To efficiently support parallelism and concurrent processing in the operating system, we use a [**internal mutability**] design pattern here: encapsulate all the mutable parts of the object into an internal object `DummyObjectInner` and wrap it in the original object with a spinlock [`Mutex`] that guarantees mutually exclusive access, leaving all the other fields immutable.
The `Mutex` takes care of the concurrent access problem in the simplest way possible: if someone else is accessing it, I'll just wait here and be busy.
Once the data is wrapped by `Mutex`, it needs to be locked first using [`lock()`] before it can be accessed. At this point concurrent access is safe, so the wrapped structure automatically has the `Send + Sync` feature.

[`Mutex`]: https://docs.rs/spin/0.5.2/spin/struct.Mutex.html
[`lock()`]: https://docs.rs/spin/0.5.2/spin/struct.Mutex.html#method.lock
[**Internal mutability**]: https://kaisery.github.io/trpl-zh-cn/ch15-05-interior-mutability.html

The use of spin locks introduces a new dependency library [`spin`] which requires the following declaration in `Cargo.toml`:

[`spin`]: https://docs.rs/spin/0.5.2/spin/index.html

``toml
[dependencies]
{{#include ... /... /code/ch01-01/Cargo.toml:spin}}
```

Then we implement the constructor for the new object:

```rust,noplaypen
// src/object/object.rs
{{#include ... /... /code/ch01-01/src/object/object_v1.rs:dummy_new}}
```

According to the documentation, each kernel object has a unique ID, so we need to implement a global ID assignment method. The method used here is to use a static variable to store the next ID to be allocated, and add `1` atomically each time it is allocated.
The ID type is `u64`, which ensures that the value space is large enough that we don't have to worry about overflow problems in our lifetime. In Zircon, IDs are allocated from 1024 onwards, and below 1024 are reserved for internal kernel use.

Also note that the return type of the `new` function is not `Self` but `Arc<Self>`, which is a uniform convention made by [ `Arc` ] to facilitate parallel processing in the future.

[ `Arc` ]: https://doc.rust-lang.org/std/sync/struct.Arc.html

Finally we implement the basic interface of `KernelObject` for it:

``rust,noplaypen
// src/object/object.rs
{{#include ... /... /code/ch01-01/src/object/object_v1.rs:dummy_impl}}
```

At this point, we have taken the first step in a long journey to implement one of the simplest functions. With implementation, there must be testing! Even the simplest code has to make sure that it behaves as we expect it to.
Only by fully testing the existing code can we be confident that we won't screw things up when we make future additions and changes. As the old saying goes, "A building is only as good as its foundation".

To prove the correctness of the above code, let's write a simple unit test replacing the self-contained `it_works` function:

``rust,noplaypen
// src/object/object.rs
{{#include ... /... /code/ch01-01/src/object/object_v1.rs:dummy_test}}
```

```sh
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.53s
     Running target/debug/deps/zcore-ae1be84852989b13

running 1 test
test tests::dummy_object ... ok
OK.

Great job! Let's format the code with the `cargo fmt` command, and remember to `git commit` to save our progress in time.

## Implementing interface to concrete type down conversion

In a system call, the user process passes in a handle to a kernel object, which the kernel then tries to convert to a specific type of object, depending on the type of the system call.
An important requirement arises here: converting the interface `Arc<dyn KernelObject>` to the concrete type structure `Arc<T> where T: KernelObject`.
This operation is called **down conversion (downcast)** in object-oriented languages.

In most programming languages, down-conversion is a very easy task. In C/C++, for example, we can write it like this

```c++
struct KernelObject {...} .
struct DummyObject: KernelObject {...} .

KernelObject *base = ... .
// C style: forced type conversion
DummyObject *dummy = (DummyObject*)(base).
// C++ style: dynamic type conversion
DummyObject *dummy = dynamic_cast<DummyObject*>(base).
```

However, in Rust, down conversion is not an easy task due to the limitations of its trait model.
Although the [`Any`] trait is provided in the standard library, which partially implements dynamic typing, it is difficult to do in practice.
For those who don't believe in it, you can toss in your own:

[``Any``]: https://doc.rust-lang.org/std/any/

```rust,editable
# use std::any::Any.
# use std::sync::Arc.
# fn main() {}

trait KernelObject: Any + Send + Sync {}
fn downcast_v1<T: KernelObject>(object: Arc<dyn KernelObject>) -> Arc<T> {
    object.downcast::<T>().unwrap()
}
fn downcast_v2<T: KernelObject>(object: Arc<dyn KernelObject>) -> Arc<T> {
    let object: Arc<dyn Any + Send + Sync + 'static> = object.
    object.downcast::<T>().unwrap()
}
```

Of course this problem has plagued a lot of people in the Rust community. A nice solution has been proposed for the [`downcast-rs`] library, which we'll introduce next:

[``downcast-rs``]: https://docs.rs/downcast-rs/1.2.0/downcast_rs/index.html

```toml
[dependencies]
{{#include ... /... /code/ch01-01/Cargo.toml:downcast}}
```

(As an aside: this library originally did not support `no_std`, and zCore had a need for it, so we helped him implement it)

As described in the [``downcast-rs``] documentation, to implement a down-conversion for our own interface, we just need to make the following changes:

``rust,noplaypen
// src/object/mod.rs
use core::fmt::Debug.
use downcast_rs::{impl_downcast, DowncastSync}.

pub trait KernelObject: DowncastSync + Debug {...}
impl_downcast!(sync KernelObject);
```

where `DowncastSync` replaces `Send + Sync` and `Debug` is used to output debugging information in case of errors.
The `impl_downcast!` macro is used to automatically generate the conversion function for us, and then we can use `downcast_arc` to do the down conversion on `Arc`. Let's test one directly:

``

## Simulating inheritance: Automatically generating interface implementation code with macros

Above we have fully implemented a kernel object and the code looks very clean. However, when we want to implement more objects, a problem arises:
These objects have some common properties and interface methods have a common implementation.
In traditional OOP languages, we usually use **inheritance** to reuse this public code: child class B can inherit from parent class A and then automatically have all the fields and methods of the parent class.

Inheritance is a very powerful feature, but its drawbacks have been discovered over time. Interested readers can take a look at the discussion on Zhihu: [* What are the disadvantages of object-oriented programming? *].
The classic work "Design Patterns" encourages people to **use combinations instead of inheritance**. And some modern programming languages, such as Go and Rust, even abandon inheritance outright. In Rust, inheritance is often partially emulated using combinatorial structures and [`Deref`] traits.

What are the disadvantages of [* object-oriented programming? *]: https://www.zhihu.com/question/20275578/answer/26577791
[`Deref`]: https://kaisery.github.io/trpl-zh-cn/ch15-02-deref.html

> Inheritance is barbaric, trait is civilized. -- some Rust enthusiast

Next we'll mimic the `downcast-rs` library and use a macro-based code generation scheme to implement `KernelObject` inheritance.
Of course, this is only a throwaway, so if readers have implemented it themselves, or if they know of a better solution in the community, they are welcome to point it out.

The approach is as follows:

- Use a struct to provide all public properties and methods as the first member of all subclasses.
- Implement the trait interface for the subclasses, and delegate all methods directly to the internal struct. This part uses macros to automatically generate the template code.

The so-called internal struct is actually the `DummyObject` we implemented above. To better reflect its functionality, let's rename it `KObjectBase`:

``rust,noplaypen
// src/object/mod.rs
{{#include ... /... /code/ch01-01/src/object/mod.rs:base_def}}
```

Next we change its constructor to implement the ``Default`` trait, and specify the public properties and methods as ``pub``:

```rust,noplaypen
// src/object/mod.rs
{{#include ... /... /code/ch01-01/src/object/mod.rs:base_default}}
impl KObjectBase {
    /// Generate a unique ID
    fn new_koid() -> KoID {...}
    /// Get the object name
    pub fn name(&self) -> String {...}
    /// Set the object name
    pub fn set_name(&self, name: &str) {...}
}
```

And finally a magic macro!

```rust,noplaypen
// src/object/mod.rs
{{#include ... /... /code/ch01-01/src/object/mod.rs:impl_kobject}}
```

The wheel is built! Let's see how we can use it to conveniently implement a kernel object, still using ``DummyObject`` as an example:

```rust,noplaypen
// src/object/mod.rs
{{#include ... /... /code/ch01-01/src/object/mod.rs:dummy}}
```

Isn't that a lot easier? Finally, as is customary, check the correctness of the implementation with unit tests: ``

```rust,noplaypen
// src/object/mod.rs
{{#include ... /... /code/ch01-01/src/object/mod.rs:dummy_test}}
```

Interested readers can continue to explore the use of the more powerful [**procedure macro (proc_macro)**] to further simplify the template code needed to implement the new kernel object.
It would be even better if the above block of code could be reduced to the following two lines:

[**procedure macro (proc_macro)**]: https://doc.rust-lang.org/proc_macro/index.html

```rust,noplaypen
#[KernelObject]
pub struct DummyObject.
```

## Summary

In this section we have implemented the core **KernelObject** concept of Zircon in Rust. A number of Rust language features and design patterns were involved in this process:

- Implementing interfaces using **trait**
- Implementing concurrent objects using the **internal mutability** pattern
- Implementing a **down conversion** of trait to struct based on community solutions
- Simulation of inheritance using combinations and automatic generation of template code using **macros**

Due to Rust's unique [object-oriented programming features], we encountered some challenges in implementing kernel objects.
But everything is difficult at the beginning, and solving these problems lays a solid foundation for the whole project, and it becomes much easier to implement new kernel objects later.

[Object-Oriented Programming Features]: https://kaisery.github.io/trpl-zh-cn/ch17-00-oop.html

In the next section, we will introduce two other concepts related to kernel objects: handles and permissions, and implement storage and access to kernel objects.
