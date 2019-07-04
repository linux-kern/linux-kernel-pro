---
layout: post
title: The block I/O latency controller
category: linux-blocks
tags: linux kernel block lwn latency cgroup
keywords: linux,kernel,block,lwn,latency,cgroup
---

{:toc}
Original&nbsp;[**Block layer discard requests**](https://lwn.net/Articles/293658/)  
Author&nbsp;&nbsp;**Jonathan Corbet**  
Date&nbsp;&nbsp;&nbsp;&nbsp;**August 12, 2008**  

**S**olid-state, flash-based storage devices are getting larger and cheaper, to the point that they are starting to displace rotating disks  
in an increasing number of systems. While flash requires less power, makes less noise, and is faster (for random reads, at least), it has  
some peculiar quirks of its own.  
One of those is the need for wear leveling - trying to keep the number of erase/write cycles on each block about the same to avoid wearing  
out the device prematurely.

Wear leveling forces the creation of an indirection layer mapping logical block numbers (as seen by the computer) to physical blocks on the media. Sometimes this mapping is done in a translation layer within the flash device itself; it can also be done within the kernel (in the UBI layer, for example) if the kernel has direct access to the flash array. Either way, this remapping comes into play anytime a block is written to the device; when that happens, a new block is chosen from a list of
free blocks and the data is written there. The block which previously contained the data is then added to the free list.
If the device fills up with data, that list of free blocks can get quite short, making it difficult to deal with writes and compromising the wear leveling algorithm. This problem is compounded by the fact that the low-level device does not really know which blocks contain useful data. You may have deleted the several hundred pieces of spam backscatter from your mailbox this morning, but the flash mapping layer has no way of knowing that, so it carefully preserves that data while
scrambling for free blocks to accommodate today's backscatter. It would be nice if the filesystem layer, which knows when the contents of files are no longer wanted, could communicate this information to the storage layer.
At the lower levels, groups like the T13 committee (which manages the ATA standards) have created protocol extensions to allow the host computer to indicate that certain sectors are no longer in use; T13 calls its new command "trim." Upon receipt of a trim command, an ATA device can immediately add the indicated sectors to its free list, discarding any data stored there. Filesystems, in turn, can cause these commands to be issued whenever a file is deleted (or truncated). That will allow the
storage device to make full use of the space which is truly free, making the whole thing work better.
What Linux lacks now, though, is the ability for filesystems to tell low-level block drivers about unneeded sectors. David Woodhouse has posted a proposal to fill that gap in the form of the discard requests patch set. As one might expect, the patches are relatively simple - there's not much to communicate - though some subtleties remain.
At the block layer, there is a new request function which can be called by filesystems:
```c
   int blkdev_issue_discard(struct block_device *bdev, sector_t sector,
              unsigned nr_sects, bio_end_io_t end_io);
```
This call will enqueue a request to bdev, saying that nr_sects sectors starting at the given  
sector are no longer needed and can be discarded. If the low-level block driver is unable to  
handle discard requests, -EOPNOTSUPP will be returned. Otherwise, the request goes onto the   
queue, and the end_io() function will be called when the discard request completes. Most of   
the time, though, the filesystem will not really care about completion - it's just passing    
advice to the driver, after all - so end_io() can beNULL and the right thing will happen.
At the driver level, a new function to set up discard requests must be provided:
```c
typedef int (prepare_discard_fn) (struct request_queue *queue, 
					struct request *req);   
void blk_queue_set_discard(struct request_queue *queue, 
	prepare_discard_fn *dfn);
```
To support discard requests, the driver should use blk_queue_set_discard() to  
register its prepare_discard_fn(). That function, in turn, will be called
whenever a discard request is enqueued; it should do whatever setup work is needed to execute this request when it gets to the head of the queue.
Since discard requests go through the queue with all other block requests, they can be manipulated by the I/O scheduler code. In particular, they can be merged, reducing the total number of requests and, perhaps, pulling together enough sectors to free a full erase block. There is a danger here, though: the filesystem may well discard a set of sectors, then write new data to them once they are allocated to a new file. It would be a serious mistake to reorder the new writes ahead of the discard operation, causing the newly-written data to be lost. So discard operations will need to function as a sort of I/O barrier, preventing the reordering of writes before and after the discard. There may be an option to drop the barrier behavior, though, for filesystems which are able to perform their own request ordering.

Outside of filesystems, there may occasionally be a need for other programs to be able to issue discard requests; David's example is mkfs, which could discard the entire contents of the device before making a new filesystem. For these applications, there is a new ioctl() call (BLKDISCARD) which creates a discard request. Needless to say, applications using this feature should be rare and very carefully written.
David's patch includes tweaks for a number of filesystems, enabling them to issue discard requests when appropriate. Some of the low-level flash drivers have been updated as well. What's missing at this point is a fix to the generic ATA driver; this will be needed to make discard requests work with flash devices using built-in translation layers - which is most of the devices on the market, currently.
That should be a relatively small piece of the puzzle, though; chances are good that this patch set will be in shape for inclusion into 2.6.28.

