---
layout: post
title: A Flowchart of Linux Block I/O
---

The Linux block I/O layer is sophisticated with a lot of subtleties, it is not practical to go through the whole layer in a post.

So this is more like a note for me, with minimal explanation.
You can check the references below to get started and go back here to see if it helps.

## Flowchart

![Flowchart](/images/bio_flowchart.png)

The states are marked with trace actions for [blkparse](https://linux.die.net/man/1/blkparse).

## References

- [Linux Device Drivers: Block Drivers](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/)
- [Understanding the Linux Kernel: Block Device Drivers](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/)
- [Block layer introduction part 1: the bio layer](https://lwn.net/Articles/736534/)
- [Block layer introduction part 2: the request layer](https://lwn.net/Articles/738449/)
- [Documentation of /sys/block/xxx/queue](https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt)
- [Documentation of /sys/block/xxx/queue/iostats](https://www.kernel.org/doc/Documentation/iostats.txt)
