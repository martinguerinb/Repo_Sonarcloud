# Security Checks

This presents a summary of the security checks introduced in glibc's implementation to detect and prevent heap related attacks.

| Function | Security Check | Error |
| --- | --- | --- |
| unlink | Whether chunk size is equal to the previous size set in the next chunk (in memory) | corrupted size vs. prev\_size |
| unlink | Whether `P->fd->bk == P` and `P->bk->fd == P`\* | corrupted double-linked list |
| \_int\_malloc | While removing the first chunk from fastbin (to service a malloc request), check whether the size of the chunk falls in fast chunk size range | malloc(): memory corruption (fast) |
| \_int\_malloc | While removing the last chunk (`victim`) from a smallbin (to service a malloc request), check whether `victim->bk->fd` and `victim` are equal | malloc(): smallbin double linked list corrupted |
| \_int\_malloc | While iterating in unsorted bin, check whether size of current chunk is within minimum (`2*SIZE_SZ`) and maximum (`av->system_mem`) range | malloc(): memory corruption |
| \_int\_malloc | While inserting last remainder chunk into unsorted bin (after splitting a large chunk), check whether `unsorted_chunks(av)->fd->bk == unsorted_chunks(av)` | malloc(): corrupted unsorted chunks |
| \_int\_malloc | While inserting last remainder chunk into unsorted bin (after splitting a fast or a small chunk), check whether `unsorted_chunks(av)->fd->bk == unsorted_chunks(av)` | malloc(): corrupted unsorted chunks 2 |
| \_int\_free | Check whether `p`\*\* is before `p + chunksize(p)` in the memory (to avoid wrapping) | free(): invalid pointer |
| \_int\_free | Check whether the chunk is at least of size `MINSIZE` or a multiple of `MALLOC_ALIGNMENT` | free(): invalid size |
| \_int\_free | For a chunk with size in fastbin range, check if next chunk's size is between minimum and maximum size (`av->system_mem`) | free(): invalid next size (fast) |
| \_int\_free | While inserting fast chunk into fastbin (at `HEAD`), check whether the chunk already at `HEAD` is not the same | double free or corruption (fasttop) |
| \_int\_free | While inserting fast chunk into fastbin (at `HEAD`), check whether size of the chunk at `HEAD` is same as the chunk to be inserted | invalid fastbin entry (free) |
| \_int\_free | If the chunk is not within the size range of fastbin and neither it is a mmapped chunks, check whether it is not the same as the top chunk | double free or corruption (top) |
| \_int\_free | Check whether next chunk (by memory) is within the boundaries of the arena | double free or corruption (out) |
| \_int\_free | Check whether next chunk's (by memory) previous in use bit is marked | double free or corruption (!prev) |
| \_int\_free | Check whether size of next chunk is within the minimum and maximum size (`av->system_mem`) | free(): invalid next size (normal) |
| \_int\_free | While inserting the coalesced chunk into unsorted bin, check whether `unsorted_chunks(av)->fd->bk == unsorted_chunks(av)` | free(): corrupted unsorted chunks |

_\*: 'P' refers to the chunk being unlinked_

_\*\*: 'p' refers to the chunk being freed_
