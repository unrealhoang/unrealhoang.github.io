+++
title = "Lock-free programming with BipBuffer"
date = "2022-05-09"
[taxonomies]
tags = ["parallel-programming", "rust", "data-struture", "parallel", "lock-free"]
+++

Lock-free programming is a term to refer to the programming of a part of a program 
(usually as a data structure / library) which coordinates the parallel processing of a shared data between
multiple CPU cores without using locks.

Let's implement a [BipBuffer](https://www.codeproject.com/Articles/3479/The-Bip-Buffer-The-Circular-Buffer-with-a-Twist) 
data structure to investigate more about the problems and solutions you will face in lock-free programming.

<!-- more -->

# BipBuffer

BipBuffer is a simple single-producer single-consumer (SPSC) ring buffer data structure that allows the producer
to reserve a portion of the queue, write to it, and commit when the writing is done. Only after committing, that
the data is available for the consumer. Similarly, the consumer can ask for what has been available, read it, then
commit the read to mark that region available for producer to write again.

Let's visualize how does it work:

at initialization, both read and write pointer is at the beginning of the buffer.
```
write:      v
Buf:       [_ _ _ _ _ _ _ _]
read:       ^
```

to write data, we must do reservation with a len, the writer will claim that space and return a mutable slice 
for us to write to. After done with writing, we commit to advance the write pointer and let the reader know 
that the before-reserved portion of the buffer is now readable.

```
to_write = write.reserve(5)

write:      v
to_write:   * * * * *
Buf:       [_ _ _ _ _ _ _ _]
read:       ^

to_write.write("ABCDE");
to_write.commit()

write:                v
Buf:       [A B C D E _ _ _]
read:       ^
```

On reader side, the reader will on request, return a non-mutable slice of available/readable data in the buffer.
Similarly, after reading is done, we commit the read to advance the read pointer and let the writer know that
portion is now available to be written again.

```
to_read = read.readable()

write:                v
Buf:       [A B C D E _ _ _]
to_read:    * * * * *
read:       ^

to_read == "ABCDE"
to_read.commit_read(5)

write:                v
Buf:       [A B C D E _ _ _]
read:                 ^
```

now let's see how the buffer behave when we are at the end of the buffer,
the reservation of len 2 is still supported by the buffer -> proceed as normal.

```
to_write = write.reserve(2)
to_write.write("XY");
to_write.commit()

write:                    v
Buf:       [A B C D E X Y _]
read:                 ^
```

at this point, the reservation of len 3 can not be made (only 1 available) at the end of the buffer,
the reservation will wrap around and give back a mutable slice of len 3 at the beginning of the buffer.

```
to_write = write.reserve(3)

write:                    v
to_write:   x x x
Buf:       [A B C D E X Y _]
read:                 ^
```

on committing the write at this point, there's one more thing we need to care about here:
we need some way to let the reader know that the last data slot in the buffer is not available to read
(since writer didn't write to that slot). We do that by introducing a new pointer of watermark. The next read
will return data from the read pointer to the watermark pointer instead of to the end of the buffer.

```
to_write.write("123");
to_write.commit()

write:            v
watermark:                v
Buf:       [1 2 3 D E X Y _]
to_read:              * *
read:                 ^

to_read = read.readable()
```

# Interface

with that visualization, we can reduce the interface of our queue as follow in Rust:

```rust
impl BipQueue {
    fn with_capacity(len: usize) -> (QueueReader, QueueWriter);
}
```

this method will create a buffer with provided capacity and return two halves (reader, writer) of the buffer


```rust
impl<'a> ReadSlice<'a> {
    fn commit_read(self, len: usize);
}
impl<'a> AsRef<[u8]> for ReadSlice<'a> {};

impl QueueReader {
    fn readable<'a>(&'a mut self) -> ReadSlice<'a>;
}
```

Reader half of the buffer, provides a `readable` function that return a slice of available read data to the caller. 
The ReadSlice is readable as `&[u8]` via the `AsRef` implementation.
The ReadSlice has a `commit_read` function that destroy the slice (`self` is passed by ownership) and update 
the read pointer in the buffer.

```rust
impl<'a> WriteSlice<'a> {
    fn commit_write(self, len: usize);
}
impl<'a> AsMut<[u8]> for WriteSlice<'a> {};

impl QueueWriter {
    fn reserve<'a>(&'a mut self, len: usize) -> Option<WriteSlice<'a>>;
}
```

Writer half of the buffer, provides a `reserve` function for the caller to specify the amount of bytes it want to 
write, if there's not enough space left in the buffer for such reservation, the `reserve` function will fail and 
return `None`. 
The ReadSlice is writable as `&mut [u8]` via the `AsMut` implementation.
Similar to the ReadSlice, the WriteSlice has a `commit_write` function that destroy the slice 
(`self` is passed by ownership) and update the write pointer in the buffer.
