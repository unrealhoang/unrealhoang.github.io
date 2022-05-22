+++
title = "Parallel programming design with BipBuffer"
date = "2022-05-09"
[taxonomies]
tags = ["parallel-programming", "rust", "unsafe", "data-struture", "parallel", "lock-free"]
+++

Coordinate the processing pattern of multiple CPU cores around shared memory data structure is one of the main
focus of parallel programming.
Let's explore the space, its problems and the iterative process to step by step find better data design,
better data access pattern to get the best out of our modern, multiple CPU cores world by implementing a
[BipBuffer](https://www.codeproject.com/Articles/3479/The-Bip-Buffer-The-Circular-Buffer-with-a-Twist)
and try to improve it.


# BipBuffer

BipBuffer is a simple single-producer single-consumer (SPSC) ring buffer data structure that allows the producer
to reserve a portion of the queue, write to it, and commit when the writing is done. Only after committing, that
the data is available for the consumer. Similarly, the consumer can ask for what has been available, read it, then
commit the read to mark that region available for producer to write again.

<!-- more -->

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
lock to coordinate between the reader & writer. Let's check the performance
with the benchmark code:

```rust
fn do_some_reading(buf: &[u8]) -> u64 {
    let mut s = 0;
    for i in 1..10 {
        for b in buf {
            s += (*b - b'0') as u64 ^ i;
        }
    }
    s
}

fn test_read_write<S: SpscQueue>(buf_size: usize, iter: usize) -> u64 {
    let (mut reader, mut writer) = S::with_len(buf_size);
    let write_thread = thread::spawn(move || {
        for i in 0..iter {
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
        let mut sum = 0;
        for i in 0..iter {
            let buf = loop {
                let r = reader.readable();
                if r.as_ref().len() >= 11 {
                    break r;
                }
            };
            sum += do_some_reading(&buf.as_ref()[0..11]);
            assert_eq!(&buf.as_ref()[0..11], b"12345678901");
            buf.commit_read(11);
        }
        sum
    });

    write_thread.join().unwrap();
    read_thread.join().unwrap()
}

fn bench_locking_bbq(c: &mut Criterion) {
    c.bench_function("locking_bbq_small_read", |b| {
        b.iter(|| {
            let r = test_read_write::<bbq::locking::Locking>(1000, 100_000, 2);
            black_box(r);
        });
    });
    c.bench_function("locking_bbq_big_read", |b| {
        b.iter(|| {
            let r = test_read_write::<bbq::locking::Locking>(1000, 100_000, 50);
            black_box(r);
        });
    });
}
```

here we have 2 thread running in parallel with 1 writer trying to write 
100K times with 11 bytes payload and reader trying to read 11 bytes payload
100K times. We split them into 2 tests: 1 where we only do a small read over
the data we received at Reader side, another for when we will loop over the data
multiple times (50) to simulate some long workload at reader.

**Result**: 
```
locking_bbq_small_read  time:   [44.760 ms 46.485 ms 48.209 ms]
locking_bbq_big_read    time:   [60.818 ms 62.834 ms 64.762 ms]
```

So as we expected, time spending at reader side directly increase the overall time result.
Let's try to improve.


## Improved lock implementation

With the previous implement we can make some observations: 

* While the `read_slice` is live, the `writer` can't do anything and vice versa. 
The execution timeline of `reader` and `writer` can be visualized as follow:
    <!--
    @startuml
    !theme mars
    participant Reader
    participant Buf
    participant Writer

    Writer -> Buf: reserve()

    Buf -> Writer: lock acquired
    activate Buf #Red

    Reader -> Buf: readable()

    Writer -> Writer: write

    Writer -> Buf: commit_write()
    deactivate Buf

    Buf -> Reader: lock acquired
    activate Buf #Blue

    Writer -> Buf: reserve()

    Reader -> Reader: read

    Reader -> Buf: commit_read()
    deactivate Buf

    Buf -> Writer: lock acquired
    activate Buf #Red
    @enduml
    -->
    ![BipBuffer lock execution timeline 1](/images/bipbuffer-lock-1.png)

* The `read_slice` and `write_slice` never overlap at any point in time:
    ```
    write_slice              [           ]
    buf          [R R R R R R W W W W W W _ _ _ _ _ _ _]
    read_slice   [           ]

    or

    write_slice                            [     ]
    buf          [_ _ _ R R R R R R _ _ _ _ W W W _ _ _]
    read_slice         [           ]

    or if the write wrapped around:

    watermark                                   v
    write_slice  [   ]
    buf          [W W _ _ _ _ R R R R R R _ _ _ _ _ _ _]
    read_slice               [           ]
    ```
  Point is, according our rule of processing (base on read pointer, write pointer and watermark pointer),
  we never hand out a slice of data that is accessible from both `writer` thread and `reader` thread.


**=>** We don't have to keep the lock to protect `read_slice` and `write_slice`. We should only lock when
trying to read or write to the pointers (`read`, `write`, `watermark`). We want to restructure our structs a bit.
Full code could be found [here](https://github.com/unrealhoang/barbequeue/blob/615ed62bb9aec21087de5af448959c769e771393/src/lock_ptr.rs)

```rust
struct BufPtrs {
    read: usize,
    write: usize,
    last: usize,
}

struct BipBuf {
    owned: Box<[u8]>,
    // length of the owned buffer
    len: usize,
    // pointer to the beginning of the owned buffer
    buf: *mut u8,
    ptrs: Mutex<BufPtrs>,
}

pub struct Reader {
    inner: Arc<BipBuf>,
}

pub struct Writer {
    inner: Arc<BipBuf>,
}

pub struct ReadSlice<'a> {
    reader: &'a mut Reader,
    range: Range<usize>,
}

pub struct WriteSlice<'a> {
    writer: &'a mut Writer,
    range: Range<usize>,
    watermark: Option<usize>,
}
```

instead of Mutex wrapping the buffer before, now we only wrap/protect the pointers. But in order to split the data
buffer into disjointed mutable slices, similar to
[split_at_mut](https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut) with custom split logic, we
must venture to the scary `unsafe` world to use the pointer and let Rust know, because there's no safe way? (if you can, please let me know)
to communicate the custom access guarantee to Rust. 
Let's change the logic code to accommodate the new structure.

```rust
impl QueueReader {
    fn readable(&mut self) -> ReadSlice<'_> {
        let lock = self.inner.ptrs.lock().unwrap();
        if lock.read <= lock.write {
            let range = lock.read..lock.write;
            drop(lock);
            ReadSlice {
                reader: self,
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
        if len > self.range.end - self.range.start {
            panic!("commit read larger than readable range");
        }
        let mut lock = self.reader.inner.ptrs.lock().unwrap();
        lock.read = self.range.start + len;
    }
}
```

So now we only lock, read the pointers, then release the lock, on commit, lock again to update the pointers, 
then release, apply the same changes to writer:

```rust
impl Writer {
    fn reserve(&mut self, len: usize) -> Option<WriteSlice<'_>> {
        let lock = self.inner.ptrs.lock().unwrap();
        let buf_len = self.inner.len;
        if lock.read <= lock.write {
            if lock.write + len < buf_len {
                let range = lock.write..lock.write + len;
                let watermark = None;
                drop(lock);
                Some(WriteSlice {
                    range,
                    watermark,
                    writer: self,
                })
            } else {
                // ...
                // other cases handling omitted
            }
        } else {
            // ...
            // other cases handling omitted
        }
    }
}

impl WriteSlice<'_> {
    fn commit_write(self, len: usize) {
        let mut lock = self.writer.inner.ptrs.lock().unwrap();
        lock.write = self.range.start + len;
        if let Some(last) = self.watermark {
            lock.last = last;
        }
    }
}
```

Done, now, observing the new benchmark result:

**Result**:
```
locking_bbq_small_read     time:   [44.760 ms 46.485 ms 48.209 ms]
locking_bbq_big_read       time:   [60.818 ms 62.834 ms 64.762 ms]

locking_bbq_ptr_small_read time:   [45.901 ms 47.905 ms 49.911 ms]
locking_bbq_ptr_big_read   time:   [41.289 ms 43.230 ms 45.257 ms]
```

The result for small read is practically not change at all, or even a little worse, because
the process for 1 cycle become: lock -> read ptr -> release -> read data -> lock -> update ptr -> release.
The extra locks & releases causing more harm than good in the small read case.
But we see an obvious improvement over the big read case, where not only the read time doesn't impact the 
result time negatively, it performs better than the small read time, this is because more time spent reading
means less contention on the lock with Writer side. 

So, on the big read case, we have improved the time by 50%.
Let's review the changed execution timeline:  
<!--
@startuml
!theme mars
participant Reader
participant Buf
participant Writer

Writer -> Buf: reserve()

Buf -> Writer: lock acquired
activate Buf #Red

Reader -> Buf: readable()

Writer -> Buf: read pointers
deactivate Buf
Buf -> Writer: write_slice

Buf -> Reader: lock acquired
activate Buf #Blue
Reader -> Buf: read pointers
deactivate Buf

par in parallel
Buf -> Reader: read_slice

Writer -> Writer: write
Reader -> Reader: read

Writer -> Buf: commit_write()
end

Buf -> Writer: lock acquired
activate Buf #Red

Reader -> Buf: commit_read()
Writer -> Buf: update pointers
deactivate Buf

Buf -> Reader: lock acquired
activate Buf #Blue
Reader -> Buf: update pointers
deactivate Buf

Writer -> Buf: reserve()

Buf -> Writer: lock acquired
activate Buf #Red
@enduml
-->
![BipBuffer lock execution timeline 2](/images/bipbuffer-lock-2.png)

We can see that the locked period of the buffer is smaller, allowing reader and writer to have
a lot more time doing execution in parallel instead of waiting for each other.
In the real usage of this library, when `read` and `write` portion is doing more (and takes more time),
the different between the first implementation and this one will be even more drastic.

## Second improvement on locking implementation

Is there any room for improvement? Let's zoom into the part of the code where the lock is acquired, we see:

* `readable()` and `reserve()` only read from the pointers, `commit_read()` and `commit_write()` only write
to the pointers.
* We could split the lock to lock each of the pointer separately, this is possible because either:
    * The `read` pointer got updated (by `commit_read()` function).
    * The `write` pointer got updated (by `commit_write()` function), or
    * The `write` and `watermark` pointer got updated at the same time (by `commit_write()` function).
    In this case, let's review the guarantee that we actually require for them by considering the cases
    where they got updated separately:
        * `write` got updated (wrapped around) and `watermark` is not:
        ```
        watermark
        write             v
        buf          [W W _ _ _ _ R R R R R R _ _ _ _ _ _ _]
        read                                  ^
        ```
        This is obviously not good, since reader see the `write` pointer wrapped around but dont see the
        position of the watermark to read to.
        * `watermark` got updated and `write` pointer is not:
        ```
        watermark                                       v
        write                                           v
        buf          [W W _ _ _ _ R R R R R R _ _ _ _ _ _ _]
        read                                  ^
        ```
        This is ok, as reader side can still get the readable slice from the `read` pointer to the `write`
        pointer.  
        So the invariant here is: 
        **`watermark` must be seen updated before `write` pointer wrap around effect is seen.**


* Reader knows the position of `read` pointer entirely (since it's the one who write them) => instead
of claiming lock to read the `read` pointer, it could keep a local value of the `read` pointer and use it.
* Similarly, Writer knows the position of `write` pointer => no lock required for reading.
* Also, we could cache the `read` pointer for Writer once we load it, same for `write` and `watermark` pointer
for Reader. Only load locked pointers when absolutely need it.

With all that, let's try to apply it:

```rust
struct BufPtrs {
    read: Mutex<usize>,
    write: Mutex<usize>,
    watermark: Mutex<usize>,
}

struct BipBuf {
    owned: Box<[u8]>,
    len: usize,
    buf: *mut u8,
    ptrs: BufPtrs,
}

pub struct Reader {
    inner: Arc<BipBuf>,
    read: usize,
}

pub struct Writer {
    inner: Arc<BipBuf>,
    write: usize,
    cache_read: usize,
}

pub struct ReadableSlice<'a> {
    reader: &'a mut Reader,
    range: Range<usize>,
}

pub struct WritableSlice<'a> {
    writer: &'a mut Writer,
    range: Range<usize>,
    watermark: Option<usize>,
}
```

the changed implementation:

```rust
impl Reader {
    fn readable(&mut self) -> ReadableSlice<'_> {
        // always lock write before lock watermark
        let lock_write = self.inner.ptrs.write.lock().unwrap();
        if self.read <= *lock_write {
            let range = self.read..*lock_write;
            drop(lock_write);
            ReadableSlice {
                reader: self,
                range,
            }
        } else {
            let lock_watermark = self.inner.ptrs.watermark.lock().unwrap();
            // read > write -> wrapped around, readable from read to watermark
            if self.read == *lock_watermark {
                let range = 0..*lock_write;
                drop(lock_watermark);
                drop(lock_write);
                ReadableSlice {
                    reader: self,
                    range,
                }
            } else {
                // ...
                // other cases handling omitted
            }
        }
    }
}

impl ReadableSlice<'_> {
    fn commit_read(mut self, len: usize) {
        let mut lock = self.reader.inner.ptrs.read.lock().unwrap();
        self.reader.read = self.range.start + len;
        *lock = self.reader.read;
    }
}
```

So Reader doesn't have to lock `read` ptr on `readable`, only lock to write
it on `commit_read()`.

```rust
impl Writer {
    fn reserve(&mut self, len: usize) -> Option<WritableSlice<'_>> {
        let buf_len = self.inner.len;
        if self.cache_read <= self.write {
            if self.write + len < buf_len {
                let range = self.write..self.write + len;
                let watermark = None;
                Some(WritableSlice {
                    range,
                    watermark,
                    writer: self,
                })
            } else {
                if len < self.cache_read {
                    let range = 0..len;
                    let watermark = Some(self.write);
                    Some(WritableSlice {
                        range,
                        watermark,
                        writer: self,
                    })
                } else {
                    let read = self.inner.ptrs.read.lock().unwrap();
                    self.cache_read = *read;
                    drop(read);

                    if len < self.cache_read {
                        let range = 0..len;
                        let watermark = Some(self.write);
                        Some(WritableSlice {
                            range,
                            watermark,
                            writer: self,
                        })
                    } else {
                        None
                    }
                }
            }
        } else {
            // ...
            // other cases handling omitted
        }
    }
}

impl WritableSlice<'_> {
    fn commit_write(self, len: usize) {
        // always lock write before lock watermark
        let mut lock_write = self.writer.inner.ptrs.write.lock().unwrap();
        if let Some(watermark) = self.watermark {
            let mut lock_watermark = self.writer.inner.ptrs.watermark.lock().unwrap();
            *lock_watermark = watermark;
        }
        self.writer.write = self.range.start + len;
        *lock_write = self.writer.write;
    }
}
```

So we try our best to use local information as much as possible before
trying to reach to the lock and doing locking.

Let's check the result:

**Result**:
```
locking_bbq_small_read           time:   [44.760 ms 46.485 ms 48.209 ms]
locking_bbq_big_read             time:   [60.818 ms 62.834 ms 64.762 ms]

locking_bbq_ptr_small_read       time:   [45.901 ms 47.905 ms 49.911 ms]
locking_bbq_ptr_big_read         time:   [41.289 ms 43.230 ms 45.257 ms]

locking_bbq_ptr_local_small_read time:   [35.653 ms 38.177 ms 40.747 ms]
locking_bbq_ptr_local_big_read   time:   [41.296 ms 43.424 ms 45.622 ms]
```

So, a medium improvement (18%) on the small read case, 
and similar performance on big read case.
