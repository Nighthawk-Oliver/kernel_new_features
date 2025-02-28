`io_uring` is a new asynchronous I/O API for Linux created by Jens Axboe from Facebook. It aims at providing an API without the limitations of the current [select(2)](http://man7.org/linux/man-pages/man2/select.2.html), [poll(2)](http://man7.org/linux/man-pages/man2/poll.2.html), [epoll(7)](http://man7.org/linux/man-pages/man7/epoll.7.html) or [aio(7)](http://man7.org/linux/man-pages/man7/aio.7.html) family of system calls, which we discussed in the previous section. Given that users of asynchronous programming models choose it in the first place for performance reasons, it makes sense to have an API that has very low performance overheads. We shall see how `io_uring` achieves this in subsequent sections.

## The io_uring interface

The very name io_uring comes from the fact that the interfaces uses ring buffers as the main interface for kernel-user space communication. While there are system calls involved, they are kept to a minimum and there is a polling mode you can use to reduce the need to make system calls as much as possible.

See also

- [Submission queue polling tutorial](https://unixism.net/loti/tutorial/sq_poll.html#sq-poll) with example program.

### The mental model

The mental model you need to construct in order to use `io_uring` to build programs that process I/O asynchronously is fairly simple.

- There are 2 ring buffers, one for submission of requests (submission queue or SQ) and the other that informs you about completion of those requests (completion queue or CQ).
- These ring buffers are shared between kernel and user space. You set these up with [`io_uring_setup()`](https://unixism.net/loti/ref-iouring/io_uring_setup.html#c.io_uring_setup) and then mapping them into user space with 2 [mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) calls.
- You tell io_uring what you need to get done (read or write a file, accept client connections, etc), which you describe as part of a submission queue entry (SQE) and add it to the tail of the submission ring buffer.
- You then tell the kernel via the [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call that you’ve added an SQE to the submission queue ring buffer. You can add multiple SQEs before making the system call as well.
- Optionally, [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) can also wait for a number of requests to be processed by the kernel before it returns so you know you’re ready to read off the completion queue for results.
- The kernel processes requests submitted and adds completion queue events (CQEs) to the tail of the completion queue ring buffer.
- You read CQEs off the head of the completion queue ring buffer. There is one CQE corresponding to each SQE and it contains the status of that particular request.
- You continue adding SQEs and reaping CQEs as you need.
- There is a [polling mode available](https://unixism.net/loti/tutorial/sq_poll.html#sq-poll), in which the kernel polls for new entries in the submission queue. This avoids the system call overhead of calling [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) every time you submit entries for processing.

See also

- [The Low-level io_uring Interface](https://unixism.net/loti/low_level.html#low-level)

## io_uring performance

Because of the shared ring buffers between the kernel and user space, io_uring can be a zero-copy system. Copying bytes around becomes necessary when system calls that transfer data between kernel and user space are involved. But since the bulk of the communication in `io_uring` is via buffers shared between the kernel and user space, this huge performance overhead is completely avoided. While system calls (and we’re used to making them a lot) may not seem like a significant overhead, in high performance applications, making a lot of them will begin to matter. Also, system calls are not as cheap as they used to be. Throw in workarounds the operating system has in place to deal with [Specter and Meltdown](https://meltdownattack.com/), we are talking non-trivial overheads. So, avoiding system calls as much as possible is a fantastic idea in high-performance applications indeed.

While using synchronous programming interfaces or even when using asynchronous programming interfaces under Linux, there is at least one system call involved in the submission of each request. In `io_uring`, you can add several requests, simply by adding multiple SQEs each describing the I/O operation you want and make a single call to io_uring_enter. For starers, that’s a win right there. But it gets better.

You can have the kernel poll and pick up your SQEs for processing as you add them to the submission queue. This avoids the [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) call you need to make to tell the kernel to pick up SQEs. For high-performance applications, this means even lesser system call overheads. See [the submission queue polling tutorial](https://unixism.net/loti/tutorial/sq_poll.html#sq-poll) for more details.

With some clever use of shared ring buffers, `io_uring` performance is really memory-bound, since in polling mode, we can do away with system calls altogether. It is important to remember that performance benchmarking is a relative process with some kind of a common point of reference. According to the [io_uring paper](https://kernel.dk/io_uring.pdf), on a reference machine, in polling mode, `io_uring` managed to clock 1.7M 4k IOPS, while [aio(7)](http://man7.org/linux/man-pages/man7/aio.7.html) manages 608k. Although much more than double, this isn’t a fair comparison since [aio(7)](http://man7.org/linux/man-pages/man7/aio.7.html) doesn’t feature a polled mode. But even when polled mode is disabled, `io_uring` hits 1.2M IOPS, close to double that of [aio(7)](http://man7.org/linux/man-pages/man7/aio.7.html).

To check the raw throughput of the `io_uring` interface, there is a no-op request type. With this, on the reference machine, `io_uring` achieves 20M messages per second. See [`io_uring_prep_nop()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_nop) for more details.

## An example using the low-level API

Writing a small program that reads files and prints them on to the console, like how the Unix `cat` utility does might be a good starting point to get your hands wet with the `io_uring` API. Please see the next chapter for one such example.

## Just use liburing

While being acquainted with the low-level `io_uring` API is most certainly a good thing, in real, serious programs you probably want to use the higher-level interface provided by liburing. Programs like [QEMU](https://qemu.org/) already use it. If liburing never existed, you’d have built some abstraction layer over the low-lever `io_uring` interface. liburing does that for you and it is a well thought-out interface as well. In short, you should probably put in some effort to understand how the low-level `io_uring` interface works, but by default you should really be using `liburing` in your programs.

While there is a reference section here for it, there are some examples based on `liburing` we’ll see in the subsequent chapters.

> 原文链接：https://unixism.net/loti/what_is_io_uring.html
