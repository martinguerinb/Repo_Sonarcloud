# Bins and Chunks

A bin is a list (doubly or singly linked list) of free (non-allocated) chunks. Bins are differentiated based on the size of chunks they contain:

1. Fast bin
2. Unsorted bin
3. Small bin
4. Large bin

Fast bins are maintained using:

```c
typedef struct malloc_chunk *mfastbinptr;

mfastbinptr fastbinsY[]; // Array of pointers to chunks
```

Unsorted, small and large bins are maintained using a single array:

```c
typedef struct malloc_chunk* mchunkptr;

mchunkptr bins[]; // Array of pointers to chunks
```

Initially, during the initialization process, small and large bins are empty.

Each bin is represented by two values in the bins array. The first one is a pointer to the 'HEAD' and the second one is a pointer to the 'TAIL' of the bin list. In the case of fast bins (singly linked list), the second value is NULL.

## Fast bins

There are 10 fast bins. Each of these bins maintains a single linked list. Addition and deletion happen from the front of this list (LIFO manner).

Each bin has chunks of the same size. The 10 bins each have chunks of sizes: 16, 24, 32, 40, 48, 56, 64, 72, 80 and 88. Sizes mentioned here include metadata as well. To store chunks, 4 fewer bytes will be available (on a platform where pointers use 4 bytes). Only the `prev_size` and `size` field of this chunk will hold meta data for allocated chunks. `prev_size` of next contiguous chunk will hold user data.

No two contiguous free fast chunks coalesce together.

## Unsorted bin

There is only 1 unsorted bin. Small and large chunks, when freed, end up in this bin. The primary purpose of this bin is to act as a cache layer (kind of) to speed up allocation and deallocation requests.

## Small bins

There are 62 small bins. Small bins are faster than large bins but slower than fast bins. Each bin maintains a doubly-linked list. Insertions happen at the 'HEAD' while removals happen at the 'TAIL' (in a FIFO manner).

Like fast bins, each bin has chunks of the same size. The 62 bins have sizes: 16, 24, ... , 504 bytes.

While freeing, small chunks may be coalesced together before ending up in unsorted bins.

## Large bins

There are 63 large bins. Each bin maintains a doubly-linked list. A particular large bin has chunks of different sizes, sorted in decreasing order (i.e. largest chunk at the 'HEAD' and smallest chunk at the 'TAIL'). Insertions and removals happen at any position within the list.

The first 32 bins contain chunks which are 64 bytes apart:

1st bin: 512 - 568 bytes  
2nd bin: 576 - 632 bytes  
.  
.

To summarize:

```
No. of Bins       Spacing between bins

64 bins of size       8  [ Small bins]
32 bins of size      64  [ Large bins]
16 bins of size     512  [ Large bins]
8 bins of size     4096  [ ..        ]
4 bins of size    32768
2 bins of size   262144
1 bin  of size what's left
```

Like small chunks, while freeing, large chunks may be coalesced together before ending up in unsorted bins.

There are two special types of chunks which are not part of any bin.

## Top chunk

It is the chunk which borders the top of an arena. While servicing 'malloc' requests, it is used as the last resort. If still more size is required, it can grow using the `sbrk` system call. The `PREV_INUSE` flag is always set for the top chunk.

## Last remainder chunk

It is the chunk obtained from the last split. Sometimes, when exact size chunks are not available, bigger chunks are split into two. One part is returned to the user whereas the other becomes the last remainder chunk.
