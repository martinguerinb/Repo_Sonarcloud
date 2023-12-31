# Double Free

Freeing a resource more than once can lead to memory leaks. The allocator's data structures get corrupted and can be exploited by an attacker. In the sample program below, a fastbin chunk will be freed twice. Now, to avoid 'double free or corruption (fasttop)' security check by glibc, another chunk will be freed in between the two frees. This implies that the same chunk will be returned by two different 'mallocs'. Both the pointers will point to the same memory address. If one of them is under the control of an attacker, he/she can modify memory for the other pointer leading to various kinds of attacks (including code executions).

Consider this sample code:

```c
a = malloc(10);     // 0xa04010
b = malloc(10);     // 0xa04030
c = malloc(10);     // 0xa04050

free(a);
free(b);  // To bypass "double free or corruption (fasttop)" check
free(a);  // Double Free !!

d = malloc(10);     // 0xa04010
e = malloc(10);     // 0xa04030
f = malloc(10);     // 0xa04010   - Same as 'd' !
```

The state of the particular fastbin progresses as:

1. 'a' freed.
  > head -> a -> tail
2. 'b' freed.
  > head -> b -> a -> tail
3. 'a' freed again.
  > head -> a -> b -> a -> tail
4. 'malloc' request for 'd'.
  > head -> b -> a -> tail      [ 'a' is returned ]
5. 'malloc' request for 'e'.
  > head -> a -> tail           [ 'b' is returned ]
6. 'malloc' request for 'f'.
  > head -> tail                [ 'a' is returned ]

Now, 'd' and 'f' pointers point to the same memory address. Any changes in one will affect the other.

Note that this particular example will not work if size is changed to one in smallbin range. With the first free, a's next chunk will set the previous in use bit as '0'. During the second free, as this bit is '0', an error will be thrown: "double free or corruption (!prev)" error.
