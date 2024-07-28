+++
title = "fsync may not be enough to guarantee data durability on Mac OS"
date = "2024-07-27T12:51:44-04:00"
description = "The behaviour of fsync on Mac OS X"
tags = ["os"]
+++

On my journey to learn about the internals of databases, I stumbled upon a quite interesting(?) fact about the behaviour of `fsync` on Mac OS.

## OS buffering lore

When you instruct your program to write data to a file, the data may go through several layers before it finally end up on the physical disk platter or flash chips of your storage device.

<div class="image">
  <img src="/images/data-buffered-flow.png" alt="Diagram on data flow from application to stable storage"/>
</div>

The data may get buffered at each layer of the stack. This is mostly done to optimize on the amount of IO operations needed to be performed. If every write has to reach the disk, that would be _extremely_ slow.

When doing `write`, depending on the types of buffering (block, line, or unbuffered), the data wouldn't reach the storage device straight away unless (1) the buffer is full, (2) `fflush` is called, or (3) the output stream itself is closed. But that would only flush the data to the kernel page cache (unless you open the file with `F_NOCACHE` or `O_DIRECT`). If you want to ensure it gets written directly to disk, you would need to use `fsync`.

The `fsync` system call is part of the POSIX standard, where it is described as a way to force a physical write of data from the buffer cache. In order to ensure that all data up to the time of the fsync has been recorded on disk and thus will survive a system crash.

## fsync on Mac OS

However, on Mac OS, its implementation doesn't provide the same guarantee as described by the POSIX standard. According to the manpage for `fsync(2)` on Mac OS:

> Note that while fsync() will flush all data from the host to the drive (i.e. the "permanent storage device"), the drive itself may not physically write the data to the platters for quite some time and it may be written in an out-of-order sequence.
>
> Specifically, if the drive loses power or the OS crashes, the application may find that only some or none of their data was written. The disk drive may also re-order the data so that later writes may be present, while earlier writes are not.

This means when `fsync` is called, the data is sent to the storage device but it does't wait until the device has actually written the data to the physical media. The drive would signal to the OS that the write operation has been done but the dirty pages may still reside in the drive's [disk buffer](https://en.wikipedia.org/wiki/Disk_buffer) for a while before it is persisted. This is done to reduce the gap in performance between the host system and the disk.

To provide the guarantee as described by the standard, Mac OS suggests using the `F_FULLFSYNC` command on the `fcntl` system call.

The manpage of `fcntl(2)` described `F_FULLFSYNC` command as:

> Does the same thing as fsync(2) then asks the drive to flush all buffered data to the permanent storage device (arg is ignored). This is currently implemented on HFS, MS-DOS (FAT), and Universal Disk Format (UDF) file systems. The operation may take quite a while to complete. Certain FireWire drives have also been known to ignore the request to flush their buffered data.

Leaving the data in the disk buffer and deferring the writes to the disk platter/flash memory at some other time can be dangerous if a system crash or power outages happen before the actual write on the disk level could even take place. The filesystem may end up with corrupted state once the system recovers. This is not a theoretical edge case as it can easily be reproduced with real world workloads and power failures.

Though, on enterprise-grade SSDs, they usually come with a capacitor that acts as a backup power source to ensure that in an event of sudden power loss, the data in the volatile write cache would still get written to the flash memory. These capacitors are designed to provide just enough power to perform the flush.

Applications that needs the highest guarantee on data integrity should use `fcntl(F_FULLFSYNC)` to ensure that the data is written to the permanent storage. This is especially important for databases so as to provide the highest durability in order to be fully ACID-compliant.

---

Google's LevelDB repository has a [commit](https://github.com/google/leveldb/commit/296de8d5b8e4e57bd1e46c981114dfbe58a8c4fa) on using `fcntl(F_FULLFSYNC)` when needing to sync data to the durable media on Apple systems. The author has also included a very insightful commit message about their learnings on introducing a fallback to `fsync` when the filesystem that it operates on have no support for `F_FULLFSYNC`.
