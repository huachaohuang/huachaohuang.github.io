---
layout: post
title: NVMe Streams and Write Lifetime Hints
---

This post introduces an NVMe feature termed *Streams Directive* and a Linux API to use the feature.

## NVMe Streams

What is Streams Directive? Here is a digest extracted from the NVMe standard:

> Directives is a mechanism to exchange information between the host and the NVM controller.
>
> Streams Directive enables the host to indicate to the controller that the specified logical blocks in a write command are part of one group of associated data.
> The controller can use this information to store related data in associated locations or for other performance enhancements.

Well, the description above is quite abstract, let me give an example on NVMe SSDs to see how this can be useful.

Suppose that we have a simple SSD with two blocks and four pages in each block, like this:

![Example 1](/images/nvme_streams_example_1.png)

One thing you should know about NAND Flash SSD is that individual pages can not be erased or overwritten, a whole block must be erased at once before its pages can be written again.
You can check [this post](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/) for more details.

In this example, our job is to write three files *File 1*, *File 2* and *File 3* to the SSD.
Each of these files contains four pages, which is 12 pages in total.

Step 1, we write File 1 and File 2 one page per file at a time, by turns.
Since the SSD has no idea about the relationship of these pages, it may place them arbitrarily.
For instance, the SSD can place them sequentially, so the 1st page of File 1 goes to Page 1, the 1st page of File 2 goes to Page 2, and the 2nd page of File 1 goes to Page 3, ..., like this:

![Example 2](/images/nvme_streams_example_2.png)

Step 2, we delete File 1 and then write File 3.
Although there are four pages containing garbage data of File 1, the SSD can not rewrite them with pages of File 3 directly due to the reason I mention above.
Instead, the SSD has to do garbage collection first.
It needs to read the blocks, keep pages of File 2, erase the blocks, and then write back pages of File 2, like this:

![Example 3](/images/nvme_streams_example_3.png)

Then the SSD can write pages of File 3 to the erased pages:

![Example 4](/images/nvme_streams_example_4.png)

OK, our job is done here, let's take a step back to see what the SSD does.
It writes 8 pages in step 1, erases 2 blocks, writes back 4 pages and then writes another 4 pages in step 2.
That is, **16 pages write and 2 blocks erase** in total.

Now let's do our job again with the Streams Directive.
This time, we assign a different stream *Stream 1*, *Stream 2*, and *Stream 3* to each of the files.

Step 1, we write pages of File 1 to Stream 1 and pages of File 2 to Stream 2.
The SSD knows that pages from the same stream are related, so it tries to place them together, like this:

![Example 5](/images/nvme_streams_example_5.png)

Step 2, we write pages of File 3 to Stream 3.
The SSD still needs to do garbage collection, but since all pages in Block 1 are invalid, it can just erase the whole block:

![Example 6](/images/nvme_streams_example_6.png)

Then the SSD writes pages of File 3 together in Block 1:

![Example 7](/images/nvme_streams_example_7.png)

That's it, let's see what the SSD does this time.
It writes 8 pages in step 1, erases 1 block, and writes another 4 pages in step 2.
That is, **12 pages write and 1 block erase** in total.

Now we can see that using the Streams Directive in this example saves us **4 pages write and 1 block erase**, in another word, **25% write and 50% erase**, which is a big win.

Sounds good, want to take advantage of this feature in your applications? Here we go.

## Write Lifetime Hints

Linux exposes the Streams Directive with the `fcntl` syscall, under the name of *write lifetime hints*.
You can check [man fcntl.2](http://man7.org/linux/man-pages/man2/fcntl.2.html) to see how to use it.

The API looks simple, it gets or sets the lifetime hint of a file.
But there is a problem, it provides 6 different lifetime hints, which are relative to each other. What's that mean?
If we want to save a file for an hour and another for a week, should we use `RWH_WRITE_LIFE_SHORT` and `RWH_WRITE_LIFE_MEDIUM`, or `RWH_WRITE_LIFE_LONG` and `RWH_WRITE_LIFE_EXTREME`?

The answer is, for NVMe SSDs, both usages are the same, as long as we use two different hints.

As the manpage says:
> There are no functional semantics implied by these flags, and different I/O classes can use the write lifetime hints in arbitrary ways, so long as the hints are used consistently.

Lifetime is an abstract concept the filesystem provides to cover the differences of underlying devices.
The truth is, the NVMe driver just assigns a different stream id to each of these hints and doesn't care about their lifetime at all.

Now I think you have no problem using this API.
In the end, this is [a real-world example](https://github.com/facebook/rocksdb/pull/3095) that uses lifetime hints and shows a good improvement.

## References

- [Multi-Stream Write SSD](https://www.samsung.com/us/labs/pdfs/2016-08-fms-multi-stream-v4.pdf)
- [Stream IDs and I/O hints](https://lwn.net/Articles/685499/)
