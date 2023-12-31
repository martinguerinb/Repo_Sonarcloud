# Introduction

This book is for understanding the structure of heap memory as well as the different kinds of exploitation techniques related to it. The material provided covers in detail the implementation of glibc's heap and related memory management functions. Next, different types of attacks are discussed.

## Prerequisites

It is assumed that the reader is unfamiliar about the internals of standard library procedures such as 'malloc' and 'free'. However, basic knowledge about 'C' and overflowing the buffer is required. These can be covered in [this](https://dhavalkapil.com/blogs/Buffer-Overflow-Exploit/) blog post.

## Setup

All the programs provided in the following sections work well with POSIX compatible machines. Only the implementation of _glibc's_ heap is discussed.
