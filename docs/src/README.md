## Concise zCore Tutorials

## Do-it-yourself OS torrents: a top-down approach

zCore is a rewrite of the Zircon microkernel in Rust, the underlying kernel in the Fuchsia OS being developed by Google.

This tutorial recreates the development process of zCore based on its real-life development history. It takes the reader step by step through the implementation of your own Zircon kernel in Rust, which will eventually be able to run native shell programs.
In the process, we will experience the design philosophy of the Zircon microkernel, how to write system software in a modern way using the Rust language, and how to integrate theory and practice in a project.

Unlike traditional OS development, zCore uses a top-down approach: first implement a working libOS in the user state based on the existing functionality of the host system, and then gradually replace the underlying implementation to "port" back to bare metal.
Then the underlying implementation is gradually replaced and "ported" back to a bare metal environment. Therefore, we focus on the overall design of the system, taking a high-level view of how the OS provides services to the user, rather than getting hung up on the underlying hardware details.

For this reason, this tutorial assumes that the reader understands basic OS concepts and principles, has experience with common Linux systems, and can write simple programs using the Rust language.
If the reader is not familiar with the OS and Rust language and wants to build an OS from scratch with a bottom-up approach, [rCore Tutorial] may be a better choice.

If you're ready, let's get started!

[rCore Tutorial]: https://rcore-os.github.io/rCore-Tutorial-deploy/
