# Secure Coding Guidelines

All of the attacks mentioned above are only possible when the writer of the code makes his/her own assumptions of the various functions provided by glibc's API. For example, developers migrating from other languages such as Java, etc. assume that it is the duty of the compiler to detect overflows during runtime.

Here, some secure coding guidelines are presented. If the software is developed keeping these in mind, it will prevent the previously mentioned attacks:

1. Use only the amount of memory asked using malloc. Make sure not to cross either boundary.
2. Free only the memory that was dynamically allocated exactly once.
3. Never access freed memory.
4. Always check the return value of malloc for `NULL`.

The above-mentioned guidelines are to be followed *strictly*. Below are some additional guidelines that will help to further prevent attacks:

1. After every free, re-assign each pointer pointing to the recently freed memory to `NULL`.
2. Always release allocated storage in error handlers.
3. Zero out sensitive data before freeing it using `memset_s` or a similar method that cannot be optimised out by the compiler.
4. Do not make any assumption regarding the positioning of the returned addresses from malloc.

Happy Coding!
