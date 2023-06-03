# Physical memory: VMO by page allocation

## Introduction

> A note: The official implementation of Zircon uses a complex and elaborate tree data structure in order to efficiently support write-time replication, but it also introduces complexity and various bugs.
> We only implement a simple version here, and leave the full implementation to the reader's own exploration.
>
> Introducing the meaning and role of commit operations

commit_page and commit_pages_with functions are used to check if a physical page frame has been allocated.

## HAL: Physical Memory Management

> Implementation of PhysFrame and simplest allocator in HAL

### kernel-hal
```rust
#[repr(C)]
pub struct PhysFrame {
	// paddr physical address
    paddr: PhysAddr,
}

impl PhysFrame {
	// Allocate physical page frames
    #[linkage = "weak"]
    #[export_name = "hal_frame_alloc"]
    pub fn alloc() -> Option<Self> {
        unimplemented!()
    }

    #[linkage = "weak"]
    #[export_name = "hal_frame_alloc_contiguous"]
    pub fn alloc_contiguous_base(_size: usize, _align_log2: usize) -> Option<PhysAddr> {
        unimplemented!()
    }

    pub fn alloc_contiguous(size: usize, align_log2: usize) -> Vec<Self> {
        PhysFrame::alloc_contiguous_base(size, align_log2).map_or(Vec::new(), |base| {
            (0..size)
                .map(|i| PhysFrame {
                    paddr: base + i * PAGE_SIZE,
                })
                .collect()
        })
    }

    pub fn alloc_zeroed() -> Option<Self> {
        Self::alloc().map(|f| {
            pmem_zero(f.addr(), PAGE_SIZE);
            f
        })
    }

    pub fn alloc_contiguous_zeroed(size: usize, align_log2: usize) -> Vec<Self> {
        PhysFrame::alloc_contiguous_base(size, align_log2).map_or(Vec::new(), |base| {
            pmem_zero(base, size * PAGE_SIZE);
            (0..size)
                .map(|i| PhysFrame {
                    paddr: base + i * PAGE_SIZE,
                })
                .collect()
        })
    }

    pub fn addr(&self) -> PhysAddr {
        self.paddr
    }

    #[linkage = "weak"]
    #[export_name = "hal_zero_frame_paddr"]
    pub fn zero_frame_addr() -> PhysAddr {
        unimplemented!()
    }
}

impl Drop for PhysFrame {
    #[linkage = "weak"]
    #[export_name = "hal_frame_dealloc"]
    fn drop(&mut self) {
        unimplemented!()
    }
}
```
### kernel-hal-unix
A page frame number can be constructed with the following code. `(PAGE_SIZE..PMEM_SIZE).step_by(PAGE_SIZE).collect()` can generate the start position of a page frame every PAGE_SIZE.
```rust
lazy_static!
    static ref AVAILABLE_FRAMES: Mutex<VecDeque<usize>> =
        Mutex::new((PAGE_SIZE..PMEM_SIZE).step_by(PAGE_SIZE).collect()).
}
```
Allocating a physical page frame is a matter of popping a page number from AVAILABLE_FRAMES via pop_front

```rust
impl PhysFrame {
    #[export_name = "hal_frame_alloc"]
    pub fn alloc() -> Option<Self> {
        let ret = AVAILABLE_FRAMES
            .lock()
            .unwrap()
            .pop_front()
            .map(|paddr| PhysFrame { paddr });
        trace!("frame alloc: {:?}", ret);
        ret
    }
    #[export_name = "hal_zero_frame_paddr"]
    pub fn zero_frame_addr() -> PhysAddr {
        0
    }
}

impl Drop for PhysFrame {
    #[export_name = "hal_frame_dealloc"]
    fn drop(&mut self) {
        trace!("frame dealloc: {:?}", self);
        AVAILABLE_FRAMES.lock().unwrap().push_back(self.paddr);
    }
}
```

## Auxiliary structures: BlockRange iterator

> Implementing BlockRange

A BlockIter iterator is used in the read and write methods of VMObjectPaged which allocates memory on a per-page basis. blockIter is mainly used to chunk a section of memory and return the information of this section each time, i.e. BlockRange.
### BlockIter
```rust
#[derive(Debug, Eq, PartialEq)]
pub struct BlockRange {
    pub block: usize.
    pub begin: usize, // start position of the address within the block
    pub end: usize, // end position of the address within the block
    pub block_size_log2: u8.
}

/// Given a range and iterate sub-range for each block
pub struct BlockIter {
    pub begin: usize,
    pub end: usize,
    pub block_size_log2: u8,
}
```
block_size_log2 is log with a base block size of 2. For example, if the block size is 4096, then block_size_log2 is 12. block is the block number.
```rust
impl BlockRange {
    pub fn len(&self) -> usize {
        self.end - self.begin
    }
    pub fn is_full(&self) -> bool {
        self.len() == (1usize << self.block_size_log2)
    }
    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
    pub fn origin_begin(&self) -> usize {
        (self.block << self.block_size_log2) + self.begin
    }
    pub fn origin_end(&self) -> usize {
        (self.block << self.block_size_log2) + self.end
    }
}

impl Iterator for BlockIter {
    type Item = BlockRange;

    fn next(&mut self) -> Option<<Self as Iterator>::Item> {
        if self.begin >= self.end {
            return None;
        }
        let block_size_log2 = self.block_size_log2;
        let block_size = 1usize << self.block_size_log2;
        let block = self.begin / block_size;
        let begin = self.begin % block_size;
		// Only the last block needs to calculate the last address in the block, the others return the size of the block directly
        let end = if block == self.end / block_size {
            self.end % block_size
        } else {
            block_size
        };
        self.begin += end - begin;
        Some(BlockRange {
            block,
            begin,
            end,
            block_size_log2,
        })
    }
}
```
## Implement VMO per page allocation

> Implement for_each_page, commit, read, write functions

The VMO structure for per-page allocation is as follows:
```rust
pub struct VMObjectPaged {
    inner: Mutex<VMObjectPagedInner>,
}

/// The mutable part of `VMObjectPaged`.
#[derive(Default)]
struct VMObjectPagedInner {
    /// Physical frames of this VMO.
    frames: Vec<PhysFrame>,
    /// Cache Policy
    cache_policy: CachePolicy,
    /// Is contiguous
    contiguous: bool,
    /// Sum of pin_count
    pin_count: usize,
    /// All mappings to this VMO.
    mappings: Vec<Weak<VmMapping>>,
}
```
VMObjectPage 有两个 new 方法
```rust
impl VMObjectPaged {
    /// Create a new VMO backing on physical memory allocated in pages.
    pub fn new(pages: usize) -> Arc<Self> {
        let mut frames = Vec::new();
        frames.resize_with(pages, || PhysFrame::alloc_zeroed().unwrap()); // 分配 pages 个页帧号，并将这些页帧号的内存清零
        Arc::new(VMObjectPaged {
            inner: Mutex::new(VMObjectPagedInner {
                frames,
                ..Default::default()
            }),
        })
    }

    /// Create a list of contiguous pages
    pub fn new_contiguous(pages: usize, align_log2: usize) -> ZxResult<Arc<Self>> {
        let frames = PhysFrame::alloc_contiguous_zeroed(pages, align_log2 - PAGE_SIZE_LOG2);
        if frames.is_empty() {
            return Err(ZxError::NO_MEMORY);
        }
        Ok(Arc::new(VMObjectPaged {
            inner: Mutex::new(VMObjectPagedInner {
                frames,
                contiguous: true,
                ..Default::default()
            }),
        }))
    }
}
```
VMObjectPaged reads and writes use a very important function for_each_page. First it constructs a BlockIter iterator, and then calls the passed-in function to read or write.
```rust
impl VMObjectPagedInner {
	/// Helper function to split range into sub-ranges within pages.
    ////
    //// ```text
    /// VMO range.
    /// |----|----|----|----|----|----|
    ////
    /// buf.
    /// [====len====]
    /// |--offset--|
    ///
    /// sub-ranges.
    /// [====]
    /// [====]
    /// [===]
    /// ```
    ///
    //// ``f`` is a function to process in-page ranges.
    /// It takes 2 arguments.
    /// * `paddr`: the start physical address of the in-page range.
    //// * `buf_range`: the range in view of the input buffer.
		fn for_each_page(
        &mut self.
        offset: usize.
        buf_len: usize.
        mut f: impl FnMut(PhysAddr, Range<usize>).
    ) {
        let iter = BlockIter {
            begin: offset.
            end: offset + buf_len.
            block_size_log2: 12.
        }.
        for block in iter {
						// Get the physical address where this block starts
            let paddr = self.frames[block.block].addr().
						// the range of the physical address of this block
            let buf_range = block.origin_begin() - offset...block.origin_end() - offset.
            f(paddr + block.begin, buf_range).
        }  
    }
}
```
read and write functions, one passed in is `kernel_hal::pmem_read` and the other is `kernel_hal::pmem_write`
```rust
impl VMObjectTrait for VMObjectPaged {
    fn read(&self, offset: usize, buf: &mut [u8]) -> ZxResult {
        let mut inner = self.inner.lock().
        if inner.cache_policy ! = CachePolicy::Cached {
            return Err(ZxError::BAD_STATE).
        }
        inner.for_each_page(offset, buf.len(), |paddr, buf_range| {
            kernel_hal::pmem_read(paddr, &mut buf[buf_range]).
        }).
        Ok(())
    }

    fn write(&self, offset: usize, buf: &[u8]) -> ZxResult {
        let mut inner = self.inner.lock().
        if inner.cache_policy ! = CachePolicy::Cached {
            return Err(ZxError::BAD_STATE).
        }
        inner.for_each_page(offset, buf.len(), |paddr, buf_range| {
            kernel_hal::pmem_write(paddr, &buf[buf_range]).
        }).
        Ok(())
    }
}
```
commit function
```rust
impl VMObjectTrait for VMObjectPaged {
		fn commit_page(&self, page_idx: usize, _flags: MMUFlags) -> ZxResult<PhysAddr> {
        let inner = self.inner.lock();
        Ok(inner.frames[page_idx].addr())
    }

    fn commit_pages_with(
        &self,
        f: &mut dyn FnMut(&mut dyn FnMut(usize, MMUFlags) -> ZxResult<PhysAddr>) -> ZxResult,
    ) -> ZxResult {
        let inner = self.inner.lock();
        f(&mut |page_idx, _| Ok(inner.frames[page_idx].addr()))
    }
}
```
## VMO Copy

> Implement the create_child function

create_child is a copy of the original VMObjectPaged content
```rust
// object/vm/vmo/paged.rs

impl VMObjectTrait for VMObjectPaged {
		fn create_child(&self, offset: usize, len: usize) -> ZxResult<Arc<dyn VMObjectTrait>> {
        assert!(page_aligned(offset));
        assert!(page_aligned(len));
        let mut inner = self.inner.lock();
        let child = inner.create_child(offset, len)?;
        Ok(child)
    }

		/// Create a snapshot child VMO.
    fn create_child(&mut self, offset: usize, len: usize) -> ZxResult<Arc<VMObjectPaged>> {
        // clone contiguous vmo is no longer permitted
        // https://fuchsia.googlesource.com/fuchsia/+/e6b4c6751bbdc9ed2795e81b8211ea294f139a45
        if self.contiguous {
            return Err(ZxError::INVALID_ARGS);
        }
        if self.cache_policy != CachePolicy::Cached || self.pin_count != 0 {
            return Err(ZxError::BAD_STATE);
        }
        let mut frames = Vec::with_capacity(pages(len));
        for _ in 0..pages(len) {
            frames.push(PhysFrame::alloc().ok_or(ZxError::NO_MEMORY)?);
        }
        for (i, frame) in frames.iter().enumerate() {
            if let Some(src_frame) = self.frames.get(pages(offset) + i) {
                kernel_hal::frame_copy(src_frame.addr(), frame.addr())
            } else {
                kernel_hal::pmem_zero(frame.addr(), PAGE_SIZE);
            }
        }
        // create child VMO
        let child = Arc::new(VMObjectPaged {
            inner: Mutex::new(VMObjectPagedInner {
                frames,
                ..Default::default()
            }),
        });
        Ok(child)
    }
}

// kernel-hal-unix/sr/lib.rs

/// Copy content of `src` frame to `target` frame
#[export_name = "hal_frame_copy"]
pub fn frame_copy(src: PhysAddr, target: PhysAddr) {
    trace!("frame_copy: {:#x} <- {:#x}", target, src);
    assert!(src + PAGE_SIZE <= PMEM_SIZE && target + PAGE_SIZE <= PMEM_SIZE);
    ensure_mmap_pmem();
    unsafe {
        let buf = phys_to_virt(src) as *const u8;
        buf.copy_to_nonoverlapping(phys_to_virt(target) as _, PAGE_SIZE);
    }
}

/// Zero physical memory at `[paddr, paddr + len)`
#[export_name = "hal_pmem_zero"]
pub fn pmem_zero(paddr: PhysAddr, len: usize) {
    trace!("pmem_zero: addr={:#x}, len={:#x}", paddr, len);
    assert!(paddr + len <= PMEM_SIZE);
    ensure_mmap_pmem();
    unsafe {
        core::ptr::write_bytes(phys_to_virt(paddr) as *mut u8, 0, len);
    }
}
```
