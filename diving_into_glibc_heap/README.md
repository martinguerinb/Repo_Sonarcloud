# Diving into glibc heap

In this section, implementation of glibc's heap management functions will be discussed in depth. The analysis was done on glibc's source code dated [27th March 2017](http://repo.or.cz/glibc.git/tree/17f487b7afa7cd6c316040f3e6c86dc96b2eec30). The source is very well documented.

Apart from the source code, the matter presented is influenced by:

* [Understanding glibc malloc](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/)
* [Understanding the heap by breaking it](https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf)

Before moving into the implementation, it is important to keep the following notes in mind:

1. Instead of `size_t`, `INTERNAL_SIZE_T` is used internally (which by default is [equal](http://repo.or.cz/glibc.git/blob/17f487b7afa7cd6c316040f3e6c86dc96b2eec30:/malloc/malloc.c#l175) to `size_t`).

2. `Alignment` is defined as `2 * (sizeof(size_t))`.

3. `MORECORE` is defined as the routine to call to obtain more memory. By default it is [defined](http://repo.or.cz/glibc.git/blob/17f487b7afa7cd6c316040f3e6c86dc96b2eec30:/malloc/malloc.c#355) as `sbrk`.

Next, we shall study the different data types used internally, bins, chunks, and internals of the different functions used.

# Additional Resources

1. r2Con2016 Glibc Heap Analysis with radare2 [video]( https://www.youtube.com/watch?v=Svm5V4leEho)
