# Internal functions

This is a list of some common functions used internally. Note that some functions are in fact defined using the `#define` directive. So, changes to call parameters are in fact retained after the call. Also, it is assumed that MALLOC\_DEBUG is not set.

## arena_get (ar_ptr, size)

Acquires an arena and locks the corresponding mutex. `ar_ptr` is set to point to the corresponding arena. `size` is just a hint as to how much memory will be required immediately.

## sysmalloc [TODO]

```c
/*
   sysmalloc handles malloc cases requiring more memory from the system.
   On entry, it is assumed that av->top does not have enough
   space to service request for nb bytes, thus requiring that av->top
   be extended or replaced.
 */
```

## void alloc_perturb (char *p, size_t n)

If `perturb_byte` (tunable parameter for malloc using `M_PERTURB`) is non-zero (by default it is 0), sets the `n` bytes pointed to by `p` to be equal to `perturb_byte` ^ 0xff.

## void free_perturb (char *p, size_t n)

If `perturb_byte` (tunable parameter for malloc using `M_PERTURB`) is non-zero (by default it is 0), sets the `n` bytes pointed to by `p` to be equal to `perturb_byte`.

## void malloc_init_state (mstate av)

```c
/*
   Initialize a malloc_state struct.

   This is called only from within malloc_consolidate, which needs
   be called in the same contexts anyway.  It is never called directly
   outside of malloc_consolidate because some optimizing compilers try
   to inline it at all call points, which turns out not to be an
   optimization at all. (Inlining it in malloc_consolidate is fine though.)
 */
```

1. For non fast bins, create empty circular linked lists for each bin.
2. Set `FASTCHUNKS_BIT` flag for `av`.
3. Initialize `av->top` to the first unsorted chunk.

## unlink(AV, P, BK, FD)

This is a defined macro which removes a chunk from a bin.

1. Check if chunk size is equal to the previous size set in the next chunk. Else, an error ("corrupted size vs. prev\_size") is thrown.
2. Check if `P->fd->bk == P` and `P->bk->fd == P`. Else, an error ("corrupted double-linked list") is thrown.
3. Adjust forward and backward pointers of neighboring chunks (in list) to facilitate removal:
   1. Set `P->fd->bk` = `P->bk`.
   2. Set `P->bk->fd` = `P->fd`.

## void malloc_consolidate(mstate av)

This is a specialized version of free().

1. Chech if `global_max_fast` is 0 (`av` not initialized) or not. If it is 0, call `malloc_init_state` with `av` as parameter and return.
2. If `global_max_fast` is non-zero, clear the `FASTCHUNKS_BIT` for `av`.
3. Iterate on the fastbin array from first to last indices:
   1. Get a lock on the current fastbin chunk and proceed if not null.
   2. If previous chunk (by memory) is not in use, call `unlink` on the previous chunk.
   3. If next chunk (by memory) is not top chunk:
      1. If next chunk is not in use, call `unlink` on the next chunk.
      2. Merge the chunk with previous, next (by memory), if any is free, and then add the consolidated chunk to the head of unsorted bin.
   4. If next chunk (by memory) was a top chunk, merge the chunks appropriately into a single top chunk.

_Note_: The check for 'in use' is done using `PREV_IN_USE` flag. Hence, other fastbin chunks won't identified as free here.
