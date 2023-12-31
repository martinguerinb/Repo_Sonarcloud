# Heap memory

## What is Heap?

Heap is a memory region allotted to every program. Unlike stack, heap memory can be dynamically allocated. This means that the program can 'request' and 'release' memory from the heap segment whenever it requires. Also, this memory is global, i.e. it can be accessed and modified from anywhere within a program and is not localized to the function where it is allocated. This is accomplished using 'pointers' to reference dynamically allocated memory which in turn leads to a small _degradation_ in performance as compared to using local variables \(on the stack\).

## Using dynamic memory

`stdlib.h` provides with standard library functions to access, modify and manage dynamic memory. Commonly used functions include **malloc** and **free**:

```c
// Dynamically allocate 10 bytes
char *buffer = (char *)malloc(10);

strcpy(buffer, "hello");
printf("%s\n", buffer); // prints "hello"

// Frees/unallocates the dynamic memory allocated earlier
free(buffer);
```

The documentation about 'malloc' and 'free' says:

* **malloc**:

  ```c
  /*
    malloc(size_t n)
    Returns a pointer to a newly allocated chunk of at least n
    bytes, or null if no space is available. Additionally, on 
    failure, errno is set to ENOMEM on ANSI C systems.

    If n is zero, malloc returns a minimum-sized chunk. (The
    minimum size is 16 bytes on most 32bit systems, and 24 or 32
    bytes on 64bit systems.)  On most systems, size_t is an unsigned
    type, so calls with negative arguments are interpreted as
    requests for huge amounts of space, which will often fail. The
    maximum supported value of n differs across systems, but is in
    all cases less than the maximum representable value of a
    size_t.
  */
  ```

* **free**:

  ```c
  /*
    free(void* p)
    Releases the chunk of memory pointed to by p, that had been
    previously allocated using malloc or a related routine such as
    realloc. It has no effect if p is null. It can have arbitrary
    (i.e., bad!) effects if p has already been freed.

    Unless disabled (using mallopt), freeing very large spaces will
    when possible, automatically trigger operations that give
    back unused memory to the system, thus reducing program
    footprint.
  */
  ```

It is important to note that these memory allocation functions are provided by the standard library. These functions provide a layer between the developer and the operating system that efficiently manages heap memory. It is the responsibility of the developer to 'free' any allocated memory after using it _exactly_ once. Internally, these functions use two system calls [sbrk](http://man7.org/linux/man-pages/man2/sbrk.2.html) and [mmap](http://man7.org/linux/man-pages/man2/mmap.2.html) to request and release heap memory from the operating system. [This](https://sploitfun.wordpress.com/2015/02/11/syscalls-used-by-malloc/) post discusses these system calls in detail.

