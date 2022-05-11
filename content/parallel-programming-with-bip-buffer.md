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

# Implementations

## Naive implementation with lock

First take, let's just make an implementation that focus on satisfy our interface and works.
**Important**: When in discovery mode, don't aim for the best possible solution even if you have some idea about it.
Use an incremental approach, make a working version with the simplest way you can think of, it will help
you focus on resolving the complexity of the unknown unknowns. Otherwise, the combinatory complexity
of your best solution and the unknowns can easily drown you.

Let's make a struct to store our Buffer:

```rust
struct BipBuf {
    buf: Box<[u8]>,

    read: usize,
    write: usize,
    watermark: usize,
}
```

Nothing special, we have the buffer as an owned slice of u8 (`Box<[u8]>`),
a read pointer, write pointer and a watermark pointer. Let's go ahead to the `QueueReader` & `QueueWriter` struct.

```rust
pub struct QueueReader {
    inner: Arc<Mutex<BipBuf>>,
}

pub struct QueueWriter {
    inner: Arc<Mutex<BipBuf>>,
}
```

Reader and Writer is just a reference-counted pointer to a shared mutex of our buffer.
Let's try to implement the `readable` function, you can read the full implementation
[here](https://github.com/unrealhoang/barbequeue/blob/6fab18eb7bbbfc576c711fdf087c67ef53fe92ea/src/locking.rs#L76-L122).

```rust
impl QueueReader {
    fn readable(&mut self) -> ReadSlice<'_> {
        let inner = self.inner.lock().unwrap();
        // if read <= write, then the readable portion is from read ptr to write ptr
        if inner.read <= inner.write {
            let range = inner.read..inner.write;
            ReadSlice {
                unlocked: inner,
                range,
            }
        } else {
            // ...
            // other cases handling omitted
        }
    }
}

impl ReadSlice<'_> {
    fn commit_read(mut self, len: usize) {
        self.unlocked.read = self.range.start + len;
    }
}
```

because our buffer is wrapped inside a mutex, for the caller of `readable` function to read the buffer's data, it
must acquire the mutex lock, read the data and then release the lock. You might miss this, but the `commit_read`
function is not only update the read pointer, but also release the lock acquired in `self.unlocked` at the end,
since it take `self` by ownership. The usage of these 2 functions will look somewhat like below:

```rust
loop {
    let read_slice = reader.readable();
    if let Ok(read_len) = do_something_with_data(&read_slice) {
        read_slice.commit_read(read_len);
    }
}
```

Here Rust with ownership feature make this very safe & easy to use this API with mutex lock.
The `read_slice` will be dropped at the `commit_read` or at the end of every loop, hence the lock will be
guaranteed to be released for other thread to access, or even this thread at the next loop. 
Whereas we must be careful to use the API correctly in other languages, for example, in Go:

```go
for {
    reader.AcquireLock()
    readSlice = reader.Readable()

    // do not keep reference to readSlice that lives out of this loop or hellfire will rain down on you
    readLen, err = doSomethingWithData(readSlice.Slice())

    if err == nil {
        readSlice.CommitRead(readLen)
    }

    reader.ReleaseLock()
}

// or embedded locking inside Readable, release lock inside CommitRead
// easier to misuse
for {
    readSlice = reader.Readable()

    // do not keep reference to readSlice that lives out of this loop or hellfire will rain down on you
    readLen, err = doSomethingWithData(readSlice.Slice())

    if err == nil {
        readSlice.CommitRead(readLen)
    } else {
        readSlice.ReleaseLock()
    }
}

// or safer but harder to use closure interface
for {
    reader.Readable(func(readSlice) {
        // do not keep reference to readSlice that lives out of this loop or hellfire will rain down on you
        readLen, err := doSomethingWithData(readSlice)

        if err == nil {
            readSlice.CommitRead(readLen)
        }
    })
}
```

or better ergonomics with Java 7's [Try Resource Block](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)

```java
while(true) {
    try (readSlice = reader.readable()) {
        // do not keep reference to readSlice that lives out of this block or hellfire will rain down on you
        int readLen = doSomethingWithData(readSlice.slice())
        readSlice.commitRead(readLen)
    }
}
```

Welp, enough language bashing and Rust praising, let's continue with Writer, full implementation
[here](https://github.com/unrealhoang/barbequeue/blob/6fab18eb7bbbfc576c711fdf087c67ef53fe92ea/src/locking.rs#L124-L191):

```rust
impl QueueWriter {
    fn reserve(&mut self, len: usize) -> Option<WriteSlice<'_>> {
        let inner = self.inner.lock().unwrap();
        if inner.read <= inner.write {
            if inner.write + len < inner.owned.len() {
                let range = inner.write..inner.write+len;
                let watermark = None;
                Some(WriteSlice {
                    unlocked: inner,
                    range,
                    watermark,
                })
            } else {
                // ...
                // other cases omitted
            }
        } else {
            // ...
            // other cases omitted
        }
    }
}

impl WriteSlice<'_> {
    fn commit_write(mut self, len: usize) {
        self.unlocked.write = self.range.start + len;
        if let Some(last) = self.watermark {
            self.unlocked.last = last;
        }
    }
}
```

typical usage of the write API:

```rust
loop {
    write_data = get_from_somewhere();
    let mut buf = loop {
        if let Some(b) = writer.reserve(len_to_write) {
            break b;
        }
    };
    buf.as_mut().copy_from_slice(write_data);
    buf.commit_write(len_to_write);
}
```

So there's not much special in this implementation as we rely on a mutex
lock to coordinate between the reader & writer. But also, its performance
is also not as good, with the benchmark code:

```rust
let (mut reader, mut writer) = S::with_len(100);
let t = Instant::now();
let write_thread = thread::spawn(move || {
    for i in 0..10_000_000 {
        let mut buf = loop {
            if let Some(b) = writer.reserve(11) {
                break b;
            }
        };
        buf.as_mut().copy_from_slice(b"12345678901");
        buf.commit_write(11);
    }
});

let read_thread = thread::spawn(move || {
    for i in 0..10_000_000 {
        let buf = loop {
            let r = reader.readable();
            if r.as_ref().len() >= 11 {
                break r;
            }
        };
        assert_eq!(&buf.as_ref()[0..11], b"12345678901");
        buf.commit_read(11);
    }
});
write_thread.join().unwrap();
read_thread.join().unwrap();
println!("elapsed: {:?}", t.elapsed());
```

here we have 2 thread running in parallel with 1 writer trying to write 
10M times with 11 bytes payload and reader trying to read 11 bytes payload
10M times.

**Result**: **4.5s** to finish.

Let's try to improve.


## Careful lock implementation

With the previous implement we can make some observations: 
* While the `read_slice` is live, the `writer` can't do anything and vice versa.
* The `read_slice` and `write_slice` never overlap at any point in time.

**=>** We don't have to keep the lock to protect `read_slice` and `write_slice`. We should only lock when
trying to read or write to the pointers 
(`read`, `write`, `watermark`).
