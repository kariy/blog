+++
title = "fsync may not be enough to guarantee data durability on Mac OS"
date = "2024-06-23T12:51:44-04:00"
description = "The behaviour of fsync on Mac OS X"
draft = true
tags = ["os"]
+++

On my journey to learn about the internals of databases, I stumbled upon a quite interesting(?) fact about the behaviour of `fsync` on Mac OS.

When you instruct your program to write data to a file, the data may go through several layers before it finally end up on the physical disk platter or flash chips of your storage device.

<div class="image">
  <img src="/images/data-buffered-flow.png" alt="Diagram on data flow from application to stable storage"/>
</div>

The data may get buffered at each layer of the stack. This is usually done by the OS to optimize on the amount of IO operations being done. If every write has to reach the disk, that would be _extremely_ slow.

When doing `write`, depending on the types of buffering (block, line, or unbuffered), the data wouldn't reach its destination straight away unless (1) the buffer is full, (2) `fflush` is called, or (3) the output stream itself is closed. But that would only flush the data to the kernel page cache (unless you open the file with `F_NOCACHE` or `O_DIRECT` on Linux). If you want to ensure it gets written directly to disk, you would have to use `fsync`.

The POSIX standard describe `fsync` as a way to force a physical write of data from the buffer cache, in order to ensure that all data up to the time of the fsync has been recorded on disk and thus will survive a system crash.

However, according to the manpage for `fsync(2)` on Mac OS X:

> Note that while fsync() will flush all data from the host to the drive (i.e. the "permanent storage device"), the drive itself may not physically write the data to the platters for quite some time and it may be written in an out-of-order sequence.
>
> Specifically, if the drive loses power or the OS crashes, the application may find that only some or none of their data was written. The disk drive may also re-order the data so that later writes may be present, while earlier writes are not.

This means when `fsync` is called on Mac OS, the data is sent to the storage device, but it does't wait until the device has actually written the data to the physical media. The dirty pages may instead reside in the device write [disk buffer](https://en.wikipedia.org/wiki/Disk_buffer) for a while before it is persisted.

Just calling `fsync` does not guarantee that the data will be persisted to the permanent storage device. To provide that guarantee, Mac OS X provides the `F_FULLFSYNC` command on the `fcntl` system call.

The manpage described the `F_FULLFSYNC` command as:

> Does the same thing as fsync(2) then asks the drive to flush all buffered data to the permanent storage device (arg is ignored). This is currently implemented on HFS, MS-DOS (FAT), and Universal Disk Format (UDF) file systems. The operation may take quite a while to complete.  Certain FireWire drives have also been known to ignore the request to flush their buffered data.

## Example comparison

Let's do a simple comparison between using `fsync` and `fcntl(F_FULLFSYNC)` to flush data.

I've written a simple Rust program that writes 1GB of data to a file in 1MB chunks. For every chunk written, we call `fsync` or `fcntl(F_FULLFSYNC)` to instruct the OS to sync the dirty pages to disk. We then inspect how long it took for both methods to finish.

```rust
fn main() -> Result<(), std::io::Error> {
    let mut file = OpenOptions::new().create(true).write(true).open("file.txt")?;

    const CHUNK_SIZE: usize = 1024 * 1024; // 1 MB
    const TOTAL_SIZE: usize = 1024 * 1024 * 1024; // 1 GB

    let mut written = 0;

    while written < TOTAL_SIZE {
        file.write_all(&[0u8; CHUNK_SIZE])?;
        written += CHUNK_SIZE;
        unsafe { libc::fsync(file.as_raw_fd()) };
    }

    Ok(())
}
```

Note: This is not exactly the most accurate and comprehensive write load test. Other factors might be affecting the provided results so they might not actually be accurate. Therefore, take the results with a grain of salt, but I think it's good enough as an example.

When we run the program above, it took about ~300ms to finish writing all 1GB.

```console
Finished `release` profile [optimized] target(s) in 0.03s
 Running `target/release/macos-fsync-test`
Time taken: 302ms
```

So now, if we replace the call to `fsync` with `fcntl(F_FULLFSYNC)` to perform the flush,

```rust
// this is the same as doing `file.sync_all()`
unsafe { libc::fcntl(file.as_raw_fd(), F_FULLFSYNC) };
```

Note: You could also just call [`File::sync_all()`](https://github.com/rust-lang/rust/blob/33422e72c8a66bdb5ee21246a948a1a02ca91674/library/std/src/sys/pal/unix/fs.rs#L1188-L1200) which just call `fcntl(F_FULLFSYNC)` under the hood.

the time it took to finish the writes is now approximately ~10 times slower (~3 seconds)!

```console
Finished `release` profile [optimized] target(s) in 0.03s
 Running `target/release/macos-fsync-test`
Time taken: 3019ms
```

The time difference is aligned with the behaviour that we expect, as described by the man pages. `fcntl(F_FULLFSYNC)` is clearly doing something more than `fsync` here. Instead of just sending the dirty pages to the storage device, it also waits until it has persisted the data to the disk platters or flash chips. it flushes the dirty pages to disk and also tells it to flush its own buffer.

Applications, such as databases, that needs the highest guarantee on data integrity, and require a strict ordering of writes should use `fcntl(F_FULLFSYNC)` to ensure that the committed data is written to the permanent storage and in the order they expect. This is especially important for databases so as to provide the highest durability in order to be fully ACID-compliant.

Google's LevelDB repository has a [commit](https://github.com/google/leveldb/commit/296de8d5b8e4e57bd1e46c981114dfbe58a8c4fa) on using `fcntl(F_FULLFSYNC)` when needing to sync data to the durable media on Apple systems. The author has also included a very insightful commit message about their learnings on introducing a fallback to `fsync` when the filesystem that it operates on have no support for `F_FULLFSYNC`.
