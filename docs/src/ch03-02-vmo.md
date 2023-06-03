# Physical Memory: VMO Objects

## Introduction to VMO

> The main features of VMOs are outlined in the documentation

Virtual Memory Objects (VMOs) represent a set of physical memory pages, or potential pages (which will be created/populated on a delayed basis as needed).

They can be mapped to the address space of a Process via [`zx_vmar_map()`](https://fuchsia.dev/docs/reference/syscalls/vmar_map), or via [`zx_vmar_unmap()`](https. //fuchsia.dev/docs/reference/syscalls/vmar_unmap) to unmap it. You can use [`zx_vmar_protect()`](https://fuchsia.dev/docs/reference/syscalls/vmar_protect) to adjust the permissions of the mapped pages.

VMO can also be read directly using [`zx_vmo_read()`](https://fuchsia.dev/docs/reference/syscalls/vmo_read) and by using [`zx_vmo_write()`](https://fuchsia.dev/) Thus, by doing one-shot operations such as "create the VMO, write the dataset to it, and then give it to another process", you can avoid the overhead of mapping them to the address space.

## Implementing the VMO object framework

> Implement the VmObject structure, which defines the VmObjectTrait interface and provides three concrete implementations of Paged, Physical, and Slice

VmObject structure

```rust
// vm/vmo/mod.rs
pub struct VmObject {
    base: KObjectBase.
    resizable: bool.
    trait_: Arc<dyn VMObjectTrait>.
    inner: Mutex<VmObjectInner>.
}

impl_kobject!(VmObject).

#[derive(Default)]
struct VmObjectInner {
    parent: Weak<VmObject>.
    children: Vec<Weak<VmObject>>.
    mapping_count: usize.
    content_size: usize.
}
```
`trait_` points to an object that implements VMObjectTrait, which consists of three concrete implementations, VMObjectPage, VMObjectPhysical, VMObjectSlice. vMObjectPaged is for allocating memory per page, VMObjectSlice is mainly for shared memory. VMObjectPhysical will not be used in zCore-Tutorial for now.  
`mapping_count` indicates how many VMARs this VmObject is mapped to.  
`content_size` is the size of the allocated physical memory.  
VmObjectTrait defines a set of methods common to VMObject*

```rust
pub trait VMObjectTrait: Sync + Send {
    /// Read memory to `buf` from VMO at `offset`.
    fn read(&self, offset: usize, buf: &mut [u8]) -> ZxResult;

    /// Write memory from `buf` to VMO at `offset`.
    fn write(&self, offset: usize, buf: &[u8]) -> ZxResult;

    /// Resets the range of bytes in the VMO from `offset` to `offset+len` to 0.
    fn zero(&self, offset: usize, len: usize) -> ZxResult;

    /// Get the length of VMO.
    fn len(&self) -> usize;

    /// Set the length of VMO.
    fn set_len(&self, len: usize) -> ZxResult;

    /// Commit a page.
    fn commit_page(&self, page_idx: usize, flags: MMUFlags) -> ZxResult<PhysAddr>;

    /// Commit pages with an external function f.
    /// the vmo is internally locked before it calls f,
    /// allowing `VmMapping` to avoid deadlock
    fn commit_pages_with(
        &self,
        f: &mut dyn FnMut(&mut dyn FnMut(usize, MMUFlags) -> ZxResult<PhysAddr>) -> ZxResult,
    ) -> ZxResult;

    /// Commit allocating physical memory.
    fn commit(&self, offset: usize, len: usize) -> ZxResult;

    /// Decommit allocated physical memory.
    fn decommit(&self, offset: usize, len: usize) -> ZxResult;

    /// Create a child VMO.
    fn create_child(&self, offset: usize, len: usize) -> ZxResult<Arc<dyn VMObjectTrait>>;

    /// Append a mapping to the VMO's mapping list.
    fn append_mapping(&self, _mapping: Weak<VmMapping>) {}

    /// Remove a mapping from the VMO's mapping list.
    fn remove_mapping(&self, _mapping: Weak<VmMapping>) {}

    /// Complete the VmoInfo.
    fn complete_info(&self, info: &mut VmoInfo);

    /// Get the cache policy.
    fn cache_policy(&self) -> CachePolicy;

    /// Set the cache policy.
    fn set_cache_policy(&self, policy: CachePolicy) -> ZxResult;

    /// Count committed pages of the VMO.
    fn committed_pages_in_range(&self, start_idx: usize, end_idx: usize) -> usize;

    /// Pin the given range of the VMO.
    fn pin(&self, _offset: usize, _len: usize) -> ZxResult {
        Err(ZxError::NOT_SUPPORTED)
    }

    /// Unpin the given range of the VMO.
    fn unpin(&self, _offset: usize, _len: usize) -> ZxResult {
        Err(ZxError::NOT_SUPPORTED)
    }

    /// Returns true if the object is backed by a contiguous range of physical memory.
    fn is_contiguous(&self) -> bool {
        false
    }

    /// Returns true if the object is backed by RAM.
    fn is_paged(&self) -> bool {
        false
    }
}
```
`read()` and `write()` for reading and writing, and `zero()` for clearing a section of memory.  
The special ones are: `fn commit_page(&self, page_idx: usize, flags: MMUFlags) -> ZxResult<PhysAddr>;`, `fn commit(&self, offset: usize, len: usize) -> ZxResult ;` and `fn commit(&self, offset: usize, len: usize) -> ZxResult;` are mainly used to allocate physical memory, because with some memory allocation policies, physical memory is not always allocated right away, so commit is needed to allocate a piece of memory.  
`pin` and `unpin` are mainly used here to increase and decrease the reference count.  
VmObject implements different new methods, the difference between them is that the objects that implement trait_ are different.

```rust
impl VmObject {
    /// Create a new VMO backing on physical memory allocated in pages.
    pub fn new_paged(pages: usize) -> Arc<Self> {
        Self::new_paged_with_resizable(false, pages)
    }

    /// Create a new VMO, which can be resizable, backing on physical memory allocated in pages.
    pub fn new_paged_with_resizable(resizable: bool, pages: usize) -> Arc<Self> {
        let base = KObjectBase::new();
        Arc::new(VmObject {
            resizable,
            trait_: VMObjectPaged::new(pages),
            inner: Mutex::new(VmObjectInner::default()),
            base,
        })
    }

    /// Create a new VMO representing a piece of contiguous physical memory.
    pub fn new_physical(paddr: PhysAddr, pages: usize) -> Arc<Self> {
        Arc::new(VmObject {
            base: KObjectBase::new(),
            resizable: false,
            trait_: VMObjectPhysical::new(paddr, pages),
            inner: Mutex::new(VmObjectInner::default()),
        })
    }

    /// Create a VM object referring to a specific contiguous range of physical frame.  
    pub fn new_contiguous(pages: usize, align_log2: usize) -> ZxResult<Arc<Self>> {
        let vmo = Arc::new(VmObject {
            base: KObjectBase::new(),
            resizable: false,
            trait_: VMObjectPaged::new_contiguous(pages, align_log2)?,
            inner: Mutex::new(VmObjectInner::default()),
        });
        Ok(vmo)
    }
}
```
A snapshot copy of a VMObject can be created by `pub fn create_child(self: &Arc<Self>, resizable: bool, offset: usize, len: usize)`.
```rust
impl VmObject {
	/// Create a child VMO.
    pub fn create_child(
        self: &Arc<Self>,
        resizable: bool,
        offset: usize,
        len: usize,
    ) -> ZxResult<Arc<Self>> {
		// Create child VmObject
        let base = KObjectBase::with_name(&self.base.name());
        let trait_ = self.trait_.create_child(offset, len)?;
        let child = Arc::new(VmObject {
            base,
            resizable,
            trait_,
            inner: Mutex::new(VmObjectInner {
                parent: Arc::downgrade(self),
                ..VmObjectInner::default()
            }),
        });
		// Add child VmObject to this VmObject
        self.add_child(&child);
        Ok(child)
    }

	/// Add child to the list
    fn add_child(&self, child: &Arc<VmObject>) {
        let mut inner = self.inner.lock();
        // Determine if this child VmObject still exists, by getting the number of strong references to the child object
        inner.children.retain(|x| x.strong_count() != 0);
		// downgrade 将 Arc 转为 Weak
        inner.children.push(Arc::downgrade(child));
    }
}
```
## HAL: Emulating physical memory with files

> Introduce mmap to introduce the idea of emulating physical memory with files
>
> Create a file and map it linearly to the process address space with mmap
>
> implement pmem_read, pmem_write

### mmap

mmap is a memory-mapped file method that maps a file or other object to the process address space, achieving a one-to-one relationship between the file's disk address and a virtual address in the process virtual address space. Once such a mapping relationship is achieved, the process can read and write to this section of memory using pointers, and the system will automatically write back dirty pages to the corresponding file disk, i.e., it is done with the file without having to call read, write, and other system calls. Conversely, changes to this area in kernel space are directly reflected in user space, allowing file sharing between processes. Therefore, creating a new file and calling mmap is actually equivalent to allocating a piece of physical memory, so we can use the file to emulate physical memory.
! [mmap.png](img/mmap.png)

### Allocate address space

Create a file for the mmap system call.

```rust
fn create_pmem_file() -> File {
    let dir = tempdir().expect("failed to create pmem dir");
    let path = dir.path().join("pmem");

    // workaround on macOS to avoid permission denied.
    // see https://jiege.ch/software/2020/02/07/macos-mmap-exec/ for analysis on this problem.
    #[cfg(target_os = "macos")]
    std::mem::forget(dir);

    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .open(&path)
        .expect("failed to create pmem file");
    file.set_len(PMEM_SIZE as u64)
        .expect("failed to resize file");
    trace!("create pmem file: path={:?}, size={:#x}", path, PMEM_SIZE);
    let prot = libc::PROT_READ | libc::PROT_WRITE;
    // Call mmap (this is not a system call) to do a two-way mapping between file and memory
    mmap(file.as_raw_fd(), 0, PMEM_SIZE, phys_to_virt(0), prot);
    file
}
```
mmap:
```rust
/// Mmap frame file `fd` to `vaddr`.
fn mmap(fd: libc::c_int, offset: usize, len: usize, vaddr: VirtAddr, prot: libc::c_int) {
    // Modify permissions according to different operating systems
	// workaround on macOS to write text section.
    #[cfg(target_os = "macos")]
    let prot = if prot & libc::PROT_EXEC != 0 {
        prot | libc::PROT_WRITE
    } else {
        prot
    };
		
	// call the mmap system call, ret is the return value
    let ret = unsafe {
        let flags = libc::MAP_SHARED | libc::MAP_FIXED;
        libc::mmap(vaddr as _, len, prot, flags, fd, offset as _)
    } as usize;
    trace!(
        "mmap file: fd={}, offset={:#x}, len={:#x}, vaddr={:#x}, prot={:#b}",
        fd,
        offset,
        len,
        vaddr,
        prot,
    );
    assert_eq!(ret, vaddr, "failed to mmap: {:?}", Error::last_os_error());
}
```
Finally, create a global variable to hold this allocated memory
```rust
lazy_static! {
    static ref FRAME_FILE: File = create_pmem_file();
}
```
### pmem_read 和 pmem_write
```rust
/// Read physical memory from `paddr` to `buf`.
#[export_name = "hal_pmem_read"]
pub fn pmem_read(paddr: PhysAddr, buf: &mut [u8]) {
    trace!("pmem read: paddr={:#x}, len={:#x}", paddr, buf.len());
    assert!(paddr + buf.len() <= PMEM_SIZE);
    ensure_mmap_pmem();
    unsafe {
        (phys_to_virt(paddr) as *const u8).copy_to_nonoverlapping(buf.as_mut_ptr(), buf.len());
    }
}

/// Write physical memory to `paddr` from `buf`.
#[export_name = "hal_pmem_write"]
pub fn pmem_write(paddr: PhysAddr, buf: &[u8]) {
    trace!("pmem write: paddr={:#x}, len={:#x}", paddr, buf.len());
    assert!(paddr + buf.len() <= PMEM_SIZE);
    ensure_mmap_pmem();
    unsafe {
        buf.as_ptr()
            .copy_to_nonoverlapping(phys_to_virt(paddr) as _, buf.len());
    }
}

/// Ensure physical memory are mmapped and accessible.
fn ensure_mmap_pmem() {
    FRAME_FILE.as_raw_fd();
}
```
`ensure_mmap_pmem()` Ensure that physical memory is mapped  
`copy_to_nonoverlapping(self, dst *mut T, count: usize)` Copy self's byte sequence into dst, source and destination are not overlapping each other. `(phys_to_virt(paddr) as *const u8).copy_to_nonoverlapping(buf.as_mut_ptr(), buf.len());` by `phys_to_virt(paddr)` convert paddr plus PMEM_BASE to virtual address, then copy the bytes inside to buf.
## Implement physical memory VMO

> Implement the VmObjectPhysical method with HAL and do unit tests
Physical memory VMO structure:
```rust
pub struct VMObjectPhysical {
    paddr: PhysAddr,
    pages: usize,
    /// Lock this when access physical memory.
    data_lock: Mutex<()>,
    inner: Mutex<VMObjectPhysicalInner>,
}

struct VMObjectPhysicalInner {
    cache_policy: CachePolicy,
}
```
What is strange here is the data_lock field, the generic type of Mutex in this field is a unit type, in fact, it is equivalent to it is no "value", it only plays the role of a lock.
```rust
impl VMObjectTrait for VMObjectPhysical {
    fn read(&self, offset: usize, buf: &mut [u8]) -> ZxResult {
        let _ = self.data_lock.lock(); // 先获取锁
        assert!(offset + buf.len() <= self.len());
        kernel_hal::pmem_read(self.paddr + offset, buf); // 对一块物理内存进行读
        Ok(())
    }
}
```
## Implement slicing VMO

> Implement VmObjectSlice and do unit tests
The parent in VMObjectSlice is used to point to an actual VMO object, e.g., VMObjectPaged, so that sharing of VMObjectPaged can be achieved through VMObjectSlice.

```rust
pub struct VMObjectSlice {
    /// Parent node.
    parent: Arc<dyn VMObjectTrait>,
    /// The offset from parent.
    offset: usize,
    /// The size in bytes.
    size: usize,
}

impl VMObjectSlice {
    pub fn new(parent: Arc<dyn VMObjectTrait>, offset: usize, size: usize) -> Arc<Self> {
        Arc::new(VMObjectSlice {
            parent,
            offset,
            size,
        })
    }

    fn check_range(&self, offset: usize, len: usize) -> ZxResult {
        if offset + len >= self.size {
            return Err(ZxError::OUT_OF_RANGE);
        }
        Ok(())
    }
}
```
VMObjectSlice implements read/write, the first step is `check_range` and the second step is to call the read/write method in parent.
```rust
impl VMObjectTrait for VMObjectSlice {
    fn read(&self, offset: usize, buf: &mut [u8]) -> ZxResult {
        self.check_range(offset, buf.len())?;
        self.parent.read(offset + self.offset, buf)
    }
}
```
