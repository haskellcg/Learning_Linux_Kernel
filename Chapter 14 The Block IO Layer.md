## **前言**
	“百年一遇”的大停电，自从搬家后久好久没这么停电了，停电之后接着停水，博客也因此没及时更新，不过两天看了不少书，书籍是人类的好朋友啊！
	接着上一篇。
## **Block Devices**
*Block devices* are hardware devices distinguished by the random access of fixed-size chunks of data, such as hard disk, floppy drivers, Blu-ray readers, and flash memory.

*Character devices* are accessed as a stream of sequential data, one byte after another.

filesystems are the lingua franca of block devices.

The complexity of block devices provides a lot of room for optimizations. The topic of this chapter is how the kernel manages block devices and their request.  This part of the kernel is known as the *block I/O layer*.

## **Anatomy of a Block Device**
**sector**: the smallest addressable unit on a block device, common size if 512 bytes. Sector is a physical property of the device

**block**: the block is an abstraction of the filesystem ---- filesystem can be accessed only in multiples of block. The kernel also requires that a block be no larger than the page size. **Therefore, block sizes are a power-of-two mutiples of the sector size and are not greater than the page size.**

## **Buffers and Buffer Heads**
Each buffer is associate with exactly one block. The buffer servers as the object that represents a disk block in memory. A single page can hold one or more blocks in memory.

The **buffer_head** structure holds all the information that the kernel needs to manipulate buffers and is defined in `<linux/buffer_head.h>`
```c
struct buffer_head{
	unsigned long           b_state;          /* buffer state flags */
	struct buffer_head      *b_this_page;     /* list of page's buffers */
	struct page             *b_page;          /* associate page */
	sector_t                b_blocknr;        /* starting block number */
	size_t                  b_size;           /* size of mapping */
	char                    *b_data;          /* pointer to data within the page */
	struct block_device     *b_bdev;          /* associate block device */
	bh_end_io_t             *b_end_io;        /* I/O completion */
	void                    *b_private;       /* reserved for b_end_io */
	struct list_head        b_assoc_buffers;  /* associated mapping */
	struct address_space    *b_assoc_map;     /* associate addresss space */
	atomic_t                b_count;          /* use count */
};
```
The purpose of a buffer head is to describe this mapping between the on-disk block and the physical in-memory buffer.

Before the 2.6 kernel, the buffer head was the unit of I/O in the kernel. But this had two primary problems:
 - the buffer head was a large and unwieldy data structure, and it was neither clean nor simple to manipulate data in terms of buffer heads.
 - when used as the container for all I/O operations, the buffer head forces the kernel to break up potential large block I/O operations into multiple buffer_head structures. This results in needless overhead and space comsumption.
 
## **The bio Structure**
The basic container for block I/O within the kernel is the bio structure. This structure represents block I/O operations that are in flight as a list of segment.

A *sengment*  is a chunk of a buffer that is contiguous in memory. 

Vector I/O such as this is callled *scatter-gather I/O*.

define in `<linux/bio.h>`
```c
struct bio{
	sector_t             bi_sector;          /* associate sector on disk */
	struct bio           *bi_next;           /* list of request */
	struct block_device  *bi_device;         /* associate block device */
	unsigned long        bi_flags;           /* status and comment flags */
	unsigned long        bi_rw;              /* read or write */
	unsigned short       bi_vcnt;            /* number of bio_vecs off */
	unsigned short       bi_idx;             /* current index in bi_io_vec */
	unsigned short       bi_phys_segments;   /* number of segments */
	unsigned int         bi_size;            /* I/O count */
	unsigned int         bi_seg_front_size;  /* size of first segment */
	unsigned int         bi_seg_end_size;    /* size of end segment */
	unsigned int         bi_max_vecs;        /* maximum bio_vecs possible */
	unsigned int         bi_comp_cpu;        /* completion CPU */
	atomic_t             bi_cnt;             /* usage counter */
	struct bio_vec       *bi_io_vec;         /* bio_vec list */
	bio_end_io_t         *bi_end_io;         /* I/O completion method */
	void                 *bi_private;        /* owner-private method */
	bio_destructor_t     *bi_destructor;     /* destructor method */
	struct bio_vec       bi_inline_vecs[0];  /* inline bio vectors */
};
```
The bi_io_vec field points to an array of bio_vec structures. These structures are used as lists of individual segments in this specific block I/O operation.

Each bio_vec is treated as a vector of the form <page, offset, len>.
```c
struct bio_vec{
	/* pointer to the physical page on which this buffer resides */
	struct page       *bv_page;
	/* the length in bytes of this buffer */
	unsigned int      bv_length;
	/* ths byte offset within the page where the buffer reside */
	unsigned int      bv_offset;
};
```
The bi_idx field is used to point to the current bio_vec in the list, which helps the block I/O layer keep track of partially completed block I/O operations. A more important usage, however, is to allow the splitting of bio_structures. 

With this feature, drives implementing a Redundant Array Of Inexpensive Disk(RAID, a hard disk setup that enables single volumes to span multiple disk for perfomence and reliability purpose) can take a single bio structure, initially intended for a single device and split it among the multiple hard drives in the RAID array. All The RAID drives needs to do is copy the bio_structure and update the bi_idx field to point to where the individual drive should start its operation.

**The concept of buffer heads is still required, however; buffers heads function as descriptors, mapping disk blocks to pages.**

## **Request Queues**
Block devices maintain *request queue* to store their pending block I/O request. 

Data structure: **request_queue** defined in `<linux/blkdev.h>`

request are added to the queue by higher-level code in the kernel, such as filesystems.

Individual request on the queue are represents by *struct request*, which is also defined in `<linux/blkdev.h>`. Each request can be composed of more than one bio structure because individual requests can operate on multiple consecutive disk blocks.

## **I/O Schedulers**
Simply sending out requests to the block devices in the order that the kernel issues them, as soon as it issues them, result in poor performance, because of **disk seek**;

So kernel performs operations called **merging** and **sorting** to greatly improve the performance of the system as a whole. The subsystem of the kernel that performs these operations is called the *I/O scheduler*.

It manages the request queue with goal of reducing seeks, which result in greater global throughput.

The goal is to minimize all seeking by keeping the disk head moving in a straight line, just like *elevator*.

## **The Linus Elevator**
The Linus Elevator performs both merging and sorting. The Linus Elevator I/O scheduler performs both *front merging* and *back merging*.

If the merge attempt fails, a possible insertion point in the queue (a location in the queue where the new request fits sectorwise between the existing request) is then sought. If one is found, the new request is inserted there. If a suitable location is not found, the request is added to tail of the queue.

Additionally, if an existing request is found in the queue that is older than a perdefined threshold, the new request is added to the tail of the queue even if it can be insertion sorted elsewhere. **This prevents many requests to nearby on-disk locations from indefinitely starving request to other locations on the disk**.

`source file: block/elevator.c`

## **The Deadline I/O Scheduler**
 - In the interesting of minimizing seeks, heavy disk I/O operations to one area of the disk can indefinitily starve request operations to another part of disk;
 - worse, the general issue of request starvation introduces a specific instance of the problem as *writes starve reads*. Writes operations can usually be committed to disk whenever kernel gets around to them, entirely asynchronous with respect to the submitting application. Read operations are quite different. Normally, when an application submits a read request, the application blocks until the request is fulfilled. That is read requests occur is synchronously with respect to the submitting application. The application does not start reading the next chunk until the previous chunk is read from disk and returned to the application.

It is tough act to provide request fairness, yet maximize global throughput.

In the Deadline I/O scheduler, each request is associate with a expiration time. By default, the expiration time is 500 milliseconds in the future for read requests and 5 seconds in the future for write requests.

![这里写图片描述](http://img.blog.csdn.net/20160420171622139)

*sorted queue* : operates similarly to *the Linus Elevator* in that it maintains a request queue sorted by physical location on disk;
*write queue && read queue* : FIFO queue;

Under normal operation, the Deadline I/O scheduler pulls requests from the head of the sorted queue into the dispatch queue.
If the request at the head of either the write FIFO queue or the read FIFO queue expires (that is, if the current time becomes greater than the expiration time associated with the queue), the deadline I/O scheduler then begin servicing requests from the FIFO queue.

Because read requests are given a substantially smaller expiration value than write requests, the Deadline I/O scheduler also works to ensure that requests do not starve read request.

`source file: block/deadline-iosched.c`

## **The Anticipatory I/O Scheduler**
Consider a system undergoing heavy write activity. 
The Deadline scheduler : the preference towards read request is a good thing, but the resulting pair of seeks (one to the location of the read request and another back to the ongoing write) is detrimental to global disk throughput.

Tha Anticipatory I/O scheduler aims to continue to provide excellent read latency, but also provide excellent global throughput.

First, the Anticipatory I/O scheduler starts with the Deadline I/O scheduler as its base. The major change is the addition of an *avticipation heuristic*.

When a read request is issued, it is handled as usual, within its usual expiration period. After the request is submited, however, the Anticipatory I/O scheduler does not immediately seek back and return to handling other request. Instead, it does absolutely nothing for a few milliseconds. In those few milliseconds, there is a good chance that the application will submit another read request.

`source file: <block/as-iosched.c>`

## **The Complete Fair Queuing I/O scheduler**
CFQ I/O scheduler is designed for specialize workload.

For example, I/O request from process foo go in foo's queues, and I/O request from process bar go in bar's queues. Within each queue, requests are coalesed with ajacent request and insertion sorted.

There is one queue for each process submitting I/O;

The CFQ I/O scheduler then services the queues round robin, plucking a configurable number of request (by deafult, four) from each queue before continuing on to the next.

**It is now the default I/O scheduler in Linux**.

## **The Noop I/O scheduler**
No sorting, but perform merging as its lone chore.

When a new request is submitted to the queue, it is coalesced with any adjacent requests.

It is intended for block devices that are truly random-access, such as flash memory cards. If a block device has little or no overhead associate with "seeking", then there is no need for insertion sorting of incoming requests.

--end