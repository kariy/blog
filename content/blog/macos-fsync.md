+++
title = "On MacOS, fsync may not be enough"
date = "2024-06-23T12:51:44-04:00"
description = "The behaviour of fsync on Mac OS X"
draft = true
tags = ["os"]
+++

According to the manual page of `fsync` on Mac OS X:

> Note that while fsync() will flush all data from the host to the drive (i.e. the "permanent storage device"), the drive itself may not physically write the data to the platters for quite some time and it may be written in an out-of-order sequence.
>
> Specifically, if the drive loses power or the OS crashes, the application may find that only some or none of their data was written. The disk drive may also re-order the data so that later writes may be present, while earlier writes are not.

When `fsync` is called, the data is sent to the storage device, but it does't wait until the device has actually written the data to the physical media. The dirty pages may instead reside in the device [disk buffer](https://en.wikipedia.org/wiki/Disk_buffer) (not to be confused with the kernel page cache) for a while before it is persisted.

Just calling `fsync` does not guarantee that the data will be persisted to the permanent storage device. To provide that guarantee, Mac OS X provides the `F_FULLFSYNC` command on the `fcntl` system call.

From its manual page, the `F_FULLFSYNC` command is described as:

> Does the same thing as fsync(2) then asks the drive to flush all buffered data to the permanent storage device (arg is ignored). This is currently implemented on HFS, MS-DOS (FAT), and Universal Disk Format (UDF) file systems. The operation may take quite a while to complete.  Certain FireWire drives have also been known to ignore the request to flush their buffered data.

Let's do a simple comparison between using `fsync` and `fcntl(F_FULLFSYNC)` to flush data.

I've written a simple Rust program that writes 1GB of dumy data to a file in 1MB chunks. For every chunk written, we call `fsync` to instruct the OS to sync the dirty pages to disk. We then measure the time it takes to finish all the writes.

```rust
fn main() -> Result<(), std::io::Error> {
    const FILENAME: &str = "test_file.txt";
    let mut file = OpenOptions::new().create(true).write(true).open(FILENAME)?;

    const CHUNK_SIZE: usize = 1024 * 1024; // 1 MB
    const TOTAL_SIZE: usize = 1024 * 1024 * 1024; // 1 GB

    let mut written = 0;
    let time = std::time::Instant::now();

    while written < TOTAL_SIZE {
        file.write_all(&[0u8; CHUNK_SIZE])?;
        written += CHUNK_SIZE;
        unsafe { libc::fsync(file.as_raw_fd()) };
    }

    println!("Time taken: {}ms", time.elapsed().as_millis());
    std::fs::remove_file(FILENAME)?;

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

Note: You could also just call [`File::sync_all()`](https://github.com/rust-lang/rust/blob/33422e72c8a66bdb5ee21246a948a1a02ca91674/library/std/src/sys/pal/unix/fs.rs#L1188-L1200) which uses `fcntl(F_FULLFSYNC)` under the hood.

the time it took to finish the writes is now approximately ~10 times slower (~3 seconds)!

```console
Finished `release` profile [optimized] target(s) in 0.03s
 Running `target/release/macos-fsync-test`
Time taken: 3019ms
```

The time difference is aligned with the behaviour that we expect, as described by the man pages. `fcntl(F_FULLFSYNC)` is clearly doing something more than `fsync` here. Instead of just sending the dirty pages to the storage device, it also waits until it has persisted the data to the disk platters or flash chips. it flushes the dirty pages to the storage device and also tells the device to flush the buffered data in the disk buffer.

Applications, such as databases, that needs the highest guarantee on data integrity, and require a strict ordering of writes should use `fcntl(F_FULLFSYNC)` to ensure that the committed data is written to the permanent storage and in the order they expect. This is especially important for databases so as to provide the highest durability in order to be fully ACID-compliant.
