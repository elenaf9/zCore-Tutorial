## Virtual Memory: VMAR Objects

## VMAR Introduction

Virtual Memory Address Regions (VMARs) provide an abstraction for managing a process's address space. A handle to a Root VMAR is provided to the process creator at process creation time. This handle refers to a VMAR that spans the entire address space. This space can be accessed via [`zx_vmar_map()`](https://fuchsia.dev/docs/reference/syscalls/vmar_map) and [`zx_vmar_allocate()`](https:/ /fuchsia.dev/docs/reference/syscalls/vmar_allocate) interfaces to divide . [`zx_vmar_allocate()`](https://fuchsia.dev/docs/reference/syscalls/vmar_allocate) can be used to generate new VMARs (called sub-regions or sub-regions) that can be used to group parts of the address space together.

## Implement the VMAR object framework

> Define VmAddressRange, VmMapping
>
> Implement create_child, map, unmap, destroy functions and do unit tests to verify address space allocation

### VmAddressRegion
```rust
pub struct VmAddressRegion {
    flags: VmarFlags.
    base: KObjectBase.
    addr: VirtAddr.
    size: usize.
    parent: Option<Arc<VmAddressRegion>>.
    page_table: Arc<Mutex<dyn PageTableTrait>>.
    /// If inner is None, this region is destroyed, all operations are invalid.
    inner: Mutex<Option<VmarInner>>.
}

#[derive(Default)]
struct VmarInner {
    children: Vec<Arc<VmAddressRegion>>.
    mappings: Vec<Arc<VmMapping>>.
}
```
Constructs a root VMAR that is owned by each process.
```rust
impl VmAddressRegion {
    /// Create a new root VMAR.
    pub fn new_root() -> Arc<Self> {
        let (addr, size) = {
            use core::sync::atomic::*.
            static VMAR_ID: AtomicUsize = AtomicUsize::new(0).
            let i = VMAR_ID.fetch_add(1, Ordering::SeqCst).
            (0x2_0000_0000 + 0x100_0000_0000 * i, 0x100_0000_0000)
        }.
        Arc::new(VmAddressRegion {
            flags: VmarFlags::ROOT_FLAGS.
            base: KObjectBase::new().
            addr.
            size.
            parent: None.
            page_table: Arc::new(Mutex::new(kernel_hal::PageTable::new())), //hal PageTable
            inner: Mutex::new(Some(VmarInner::default())).
        })
    }
}
```
Our kernel also requires a root VMAR
```rust
/// The base of kernel address space
/// In x86 fuchsia this is 0xffff_ff80_0000_0000 instead
pub const KERNEL_ASPACE_BASE: u64 = 0xffff_ff02_0000_0000;
/// The size of kernel address space
pub const KERNEL_ASPACE_SIZE: u64 = 0x0000_0080_0000_0000;
/// The base of user address space
pub const USER_ASPACE_BASE: u64 = 0;
// pub const USER_ASPACE_BASE: u64 = 0x0000_0000_0100_0000;
/// The size of user address space
pub const USER_ASPACE_SIZE: u64 = (1u64 << 47) - 4096 - USER_ASPACE_BASE;

impl VmAddressRegion {
		/// Create a kernel root VMAR.
    pub fn new_kernel() -> Arc<Self> {
        let kernel_vmar_base = KERNEL_ASPACE_BASE as usize;
        let kernel_vmar_size = KERNEL_ASPACE_SIZE as usize;
        Arc::new(VmAddressRegion {
            flags: VmarFlags::ROOT_FLAGS,
            base: KObjectBase::new(),
            addr: kernel_vmar_base,
            size: kernel_vmar_size,
            parent: None,
            page_table: Arc::new(Mutex::new(kernel_hal::PageTable::new())),
            inner: Mutex::new(Some(VmarInner::default())),
        })
    }
}
```
### VmAddressMapping
VmAddressMapping is used to establish the mapping between VMO and VMAR.
```rust
/// Virtual Memory Mapping
pub struct VmMapping {
    /// The permission limitation of the vmar
    permissions: MMUFlags,
    vmo: Arc<VmObject>,
    page_table: Arc<Mutex<dyn PageTableTrait>>,
    inner: Mutex<VmMappingInner>,
}

#[derive(Debug, Clone)]
struct VmMappingInner {
    /// The actual flags used in the mapping of each page
    flags: Vec<MMUFlags>,
    addr: VirtAddr,
    size: usize,
    vmo_offset: usize,
}
```
map and unmap implement memory mapping and unmapping VmAddressMapping is used to establish the mapping between VMO and VMAR.
```rust
impl VmMapping {
	/// Mapping scopes and commits.
    /// Submit pages to vmo and map those pages to frames in page_table.
    /// Temporarily used for development. one of the standard procedures for vmo is
    /// A standard procedure for vmo is: create_vmo, op_range(commit), map
    fn map(self: &Arc<Self>) -> ZxResult {
        self.vmo.commit_pages_with(&mut |commit| {
            let inner = self.inner.lock();
            let mut page_table = self.page_table.lock();
            let page_num = inner.size / PAGE_SIZE;
            let vmo_offset = inner.vmo_offset / PAGE_SIZE;
            for i in 0... .page_num {
                let paddr = commit(vmo_offset + i, inner.flags[i]) ?
                // page table mapping via PageTableTrait's hal_pt_map
                // call kernel-hal's method for mapping.
            }
            Ok(())
        })
    }

    fn unmap(&self) {
        let inner = self.inner.lock();
        let pages = inner.size / PAGE_SIZE;
        // TODO inner.vmo_offset not used?
        // Call kernel-hal's method for unmapping.
    }
}
```
## HAL: simulate page table with mmap

> Implement page table interface map, unmap, protect

A page table and the methods this page table has are defined in kernel-hal.
```rust
/// Page Table
#[repr(C)]
pub struct PageTable {
    table_phys: PhysAddr,
}

impl PageTable {
    /// Get current page table
    #[linkage = "weak"]
    #[export_name = "hal_pt_current"]
    pub fn current() -> Self {
        unimplemented!()
    }

    /// Create a new `PageTable`.
    #[allow(clippy::new_without_default)]
    #[linkage = "weak"]
    #[export_name = "hal_pt_new"]
    pub fn new() -> Self {
        unimplemented!()
    }
}

impl PageTableTrait for PageTable {
    /// Map the page of `vaddr` to the frame of `paddr` with `flags`.
    #[linkage = "weak"]
    #[export_name = "hal_pt_map"]
    fn map(&mut self, _vaddr: VirtAddr, _paddr: PhysAddr, _flags: MMUFlags) -> Result<()> {
        unimplemented!()
    }
    /// Unmap the page of `vaddr`.
    #[linkage = "weak"]
    #[export_name = "hal_pt_unmap"]
    fn unmap(&mut self, _vaddr: VirtAddr) -> Result<()> {
        unimplemented!()
    }
    /// Change the `flags` of the page of `vaddr`.
    #[linkage = "weak"]
    #[export_name = "hal_pt_protect"]
    fn protect(&mut self, _vaddr: VirtAddr, _flags: MMUFlags) -> Result<()> {
        unimplemented!()
    }
    /// Query the physical address which the page of `vaddr` maps to.
    #[linkage = "weak"]
    #[export_name = "hal_pt_query"]
    fn query(&mut self, _vaddr: VirtAddr) -> Result<PhysAddr> {
        unimplemented!()
    }
    /// Get the physical address of root page table.
    #[linkage = "weak"]
    #[export_name = "hal_pt_table_phys"]
    fn table_phys(&self) -> PhysAddr {
        self.table_phys
    }

    /// Activate this page table
    #[cfg(target_arch = "riscv64")]
    #[linkage = "weak"]
    #[export_name = "hal_pt_activate"]
    fn activate(&self) {
        unimplemented!()
    }

    #[linkage = "weak"]
    #[export_name = "hal_pt_unmap_cont"]
    fn unmap_cont(&mut self, vaddr: VirtAddr, pages: usize) -> Result<()> {
        for i in 0..pages {
            self.unmap(vaddr + i * PAGE_SIZE)?;
        }
        Ok(())
    }
}
```
Implemented PageTableTrait in kernel-hal-unix, and called mmap in map.
```rust
implant PageTableTrait for PageTable {
    /// Mapping pages from `vaddr` to `paddr` frames with `flags`.
    #[export_name = "hal_pt_map"]
    fn map(&mut self, vaddr: VirtAddr, paddr: PhysAddr, flags: MMUFlags) -> Result<()> {
        debug_assert! (page_aligned(vaddr));
        debug_assert! (page_aligned(paddr));
        let prot = flags.to_mmap_prot();
        mmap(FRAME_FILE.as_raw_fd(), paddr, PAGE_SIZE, vaddr, prot);
        Ok(())
    }

    /// Unmapping the `vaddr` page.
    #[export_name = "hal_pt_unmap"]
    fn unmap(&mut self, vaddr: VirtAddr) -> Result<()> {
        self.unmap_cont(vaddr, 1)
    }
}
```
## Implementing memory mapping

> Use hal to implement the part of vmar left empty above, and do unit tests to verify the memory mapping
```rust
impl VmMapping {
	/// Mapping scope and commit.
    /// Submit pages to vmo and map them to frames in page_table.
    /// Temporarily used for development. one of the standard procedures for vmo is
    /// A standard procedure for vmo is: create_vmo, op_range(commit), map
    fn map(self: &Arc<Self>) -> ZxResult {
        self.vmo.commit_pages_with(&mut |commit| {
            let inner = self.inner.lock();
            let mut page_table = self.page_table.lock();
            let page_num = inner.size / PAGE_SIZE;
            let vmo_offset = inner.vmo_offset / PAGE_SIZE;
            for i in 0... .page_num {
                let paddr = commit(vmo_offset + i, inner.flags[i]) ?
                // page table mapping via PageTableTrait's hal_pt_map
                page_table
                    .map(inner.addr + i * PAGE_SIZE, paddr, inner.flags[i])
                    .expect(" failed to map");
            }
            Ok(())
        })
    }

    fn unmap(&self) {
        let inner = self.inner.lock();
        let pages = inner.size / PAGE_SIZE;
        // TODO inner.vmo_offset not used?
        self.page_table
            .lock()
            .unmap_cont(inner.addr, pages)
            .expect("failed to unmap")
    }
}
```
