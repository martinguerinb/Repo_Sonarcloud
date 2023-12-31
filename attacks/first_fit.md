# First-fit behavior

This technique describes the 'first-fit' behavior of glibc's allocator. Whenever any chunk (not a fast chunk) is freed, it ends up in the `unsorted` bin. Insertion happens at the `HEAD` of the list. On requesting new chunks (again, non fast chunks), initially unsorted bins will be looked up as small bins will be empty. This lookup is from the `TAIL` end of the list. If a single chunk is present in the unsorted bin, an exact check is not made and if the chunk's size >= the one requested, it is split into two and returned. This ensures first in first out behavior.

Consider the sample code:

```c
char *a = malloc(300);    // 0x***010
char *b = malloc(250);    // 0x***150

free(a);

a = malloc(250);          // 0x***010
```

The state of unsorted bin progresses as:

1. 'a' freed.
  > head -> a -> tail
2. 'malloc' request.
  > head -> a2 -> tail [ 'a1' is returned ]

'a' chunk is split into two chunks 'a1' and 'a2' as the requested size (250 bytes) is smaller than the size of the chunk 'a' (300 bytes). This corresponds to [6. iii.] in `_int_malloc`.

This is also true in the case of fast chunks. Instead of 'freeing' into `unsorted` bin, fast chunks end up in `fastbins`. As mentioned earlier, `fastbins` maintain a singly linked list and chunks are inserted and deleted from the `HEAD` end. This 'reverses' the order of chunks obtained.

Consider the sample code:

```c
char *a = malloc(20);     // 0xe4b010
char *b = malloc(20);     // 0xe4b030
char *c = malloc(20);     // 0xe4b050
char *d = malloc(20);     // 0xe4b070

free(a);
free(b);
free(c);
free(d);

a = malloc(20);           // 0xe4b070
b = malloc(20);           // 0xe4b050
c = malloc(20);           // 0xe4b030
d = malloc(20);           // 0xe4b010
```

The state of the particular fastbin progresses as:

1. 'a' freed.
  > head -> a -> tail
2. 'b' freed.
  > head -> b -> a -> tail
3. 'c' freed.
  > head -> c -> b -> a -> tail
4. 'd' freed.
  > head -> d -> c -> b -> a -> tail
5. 'malloc' request.
  > head -> c -> b -> a -> tail [ 'd' is returned ]
6. 'malloc' request.
  > head -> b -> a -> tail      [ 'c' is returned ]
7. 'malloc' request.
  > head -> a -> tail           [ 'b' is returned ]
8. 'malloc' request.
  > head -> tail                [ 'a' is returned ]

The smaller size here (20 bytes) ensured that on freeing, chunks went into `fastbins` instead of the `unsorted` bin.

## Use after Free Vulnerability

In the above examples, we see that, malloc _might_ return chunks that were earlier used and freed. This makes using freed memory chunks vulnerable. Once a chunk has been freed, it **should** be assumed that the attacker can now control the data inside the chunk. That particular chunk should never be used again. Instead, always allocate a new chunk.

See sample piece of vulnerable code:

```c
char *ch = malloc(20);

// Some operations
//  ..
//  ..

free(ch);

// Some operations
//  ..
//  ..

// Attacker can control 'ch'
// This is vulnerable code
// Freed variables should not be used again
if (*ch=='a') {
  // do this
}
```
