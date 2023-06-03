# zCore Tutorial

[! [CI](https://github.com/rcore-os/zCore-Tutorial/workflows/CI/badge.svg?branch=master)](https://github.com/rcore-os/zCore-Tutorial/ actions)
[! [Docs](https://img.shields.io/badge/docs-alpha-blue)](https://rcore-os.github.io/zCore-Tutorial/)

The goal of zCore Toturial is to learn and master the core concepts and corresponding implementations of the zCore Kernel by `step by step` building a simplified zCore kernel, thus laying the foundation for further analysis and mastery of the complete zCore kernel.

The zCore Toturial features all code running in the user state, making it easy to debug and analyze.

## repository directory

* `docs/`: teaching lab guide
* `code`: operating system code

## Experimental guide

Based on mdBook, currently deployed on [GitHub Pages](https://rcore-os.github.io/zCore-Tutorial/) at the moment.

### Documentation for local use

```bash
git clone https://github.com/rcore-os/zCore-Tutorial.git
cd zCore-Tutorial
cargo install mdbook
mdbook serve docs
```

## code
The contents of `rust-toolchain` in the `code` directory is `nightly-2021-07-27`. In principle, we will use the latest version of `rustc`. The current version information is as follows:
```
rustc 1.56.0-nightly (08095fc1f 2021-07-26)
```

## Suggested learning sequence

### Preliminary understanding

1. read overview/introduction articles about fuchsia/zircon, e.g. https://zh.wikipedia.org/zh-hans/Google_Fuchsia

2. read https://fuchsia.dev/fuchsia-src/concepts/kernel for basic zircon ideas

3. read the first two chapters of Qinglin Pan's thesis to understand the basic ideas of zCore

### Getting deeper
1. Read https://fuchsia.dev/fuchsia-src/reference/syscalls to understand the application requirements for Kernel
2. read https://fuchsia.dev/fuchsia-src/reference/kernel_objects/objects to understand the meaning and behavior of various objects in Kernel

### Understand the design implementation

1. read & analyze the documentation and code in this project, and compare the kernel concepts above to understand the correspondence between kernel concepts and design implementation

### Hands-on practice

1. improve the documentation of the corresponding section of the project based on the analysis and understanding

2. based on the analysis and understanding, improve/optimize the code of this project, add test cases, and increase the functionality

3. After mastering the project in general, have a good understanding of new operating systems such as zCore by further understanding and improving zCore, and improve your practical skills

   

Tips related to #### code/ch04-xx
  - Recommended way to run: In the `ch04-0x` directory: `RUST_LOG=info cargo run -p zircon-loader -- /prebuilt/zircon/x64`
  - ch4 will execute the userboot program in zircon prebuilt, see [userboot source code](https://github.com/vsrinivas/fuchsia/tree/master/zircon/kernel/lib/userabi/ userboot), [fuchsia boot process](https://fuchsia.dev/fuchsia-src/concepts/booting/userboot?hl=en).
  - `ch04-01` does not implement any syscall, so entering userboot will panic when the first syscall returns to the kernel state. 
  - `ch04-03` implements some syscalls related to `channel` and `debuglog`, and will execute 3 syscalls and then exit because process_exit is not supported.



## Reference

- https://fuchsia.dev/
  - https://fuchsia.dev/fuchsia-src/concepts/kernel
  - https://fuchsia.dev/fuchsia-src/reference/kernel_objects/objects
  - https://fuchsia.dev/fuchsia-src/reference/syscalls
  - https://github.com/zhangpf/fuchsia-docs-zh-CN/tree/master/zircon
  - [Presentation by Dr. Zhongxing Xu: Introduction to Fuchsia OS](https://xuzhongxing.github.io/201806fuchsia.pdf)
  
- Thesis
  - [Design and Implementation of Rust Language Operating System,王润基 B.S. Thesis,2019](https://github.com/rcore-os/zCore/wiki/files/wrj-thesis.pdf) 
  - [Design and implementation of zCore OS kernel,Pan Qinglin's undergraduate thesis,2020](https://github.com/rcore-os/zCore/wiki/files/pql-thesis.pdf)
  
- Development Document
  - https://github.com/rcore-os/zCore/wiki/documents-of-zcore

- The simpler and more basic [rCore-Tutorial v3](https://rcore-os.github.io/rCore-Tutorial-Book-v3/): if you can't read the above, you can read this tutorial first.
