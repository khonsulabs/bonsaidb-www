+++
title = "Optimizing BonsaiDb: Promising progress on new storage layer (Part 2 of ?)"

[extra]
author = "Jonathan Johnson"
author_url = "https://github.com/ecton"
+++

{{ whatis() }}

At the end of the [last post in this ongoing
series](/blog/optimizing-bonsaidb-p1), I introduced an overview of
[Sediment][sediment], the new storage format I was beginning development on that
I hoped would replace the append-only file format that [Nebari][nebari] uses.

It's been a month, and as promised, I wanted to share my progress. Sediment is
still very early in development. All results in this post should be taken with
healthy skepticism, as there could be bugs that I have not discovered yet that
force me to change some fundamental aspect of the library. I hope I'm beyond
those hurdles, but there are some fun and complex problems I have yet to tackle.

This post is going to be an exploration of Sediment's design. At the end, I will
show some of the early benchmarking results -- including running the [Nebari
benchmark suite][nebari-benchmarks].

## {{ anchor(text = "The Goals of Sediment" )}}

The primary speed issues I had identified with [Nebari][nebari] stemmed from two
design decisions:

- Nebari did not preallocate disk space. Allocating a larger chunk of space and
  overwriting it is faster than appending multiple times to the end of a file
  due to extra filesystem metadata syncing being required each time the file's
  length changes.
- Nebari required two `fsync()` operations for ACID. My early benchmarking was
  flawed due to using temporary files in `/tmp` for benchmarking against. On
  Linux these days, `/tmp` is often powered by the `tmpfs` filesystem. This
  filesystem does not actually persist data to disk, which means `fsync()` was
  almost instantaneous. The two-`fsync()` commit strategy is simply not viable
  if the goal is to be competitive with performance.

One of the other problems an append-only file format has is the requirement of
compaction. As a former heavy user of [CouchDB][couchdb] (the original design
inspiration for BonsaiDb), I remember struggling to compact databases when I was
running low on disk space. The problem with compacting a database is that it
requires rewriting the database to a new file -- an operation that requires free
disk space. For large databases, that could be a challenge.

I had goals for BonsaiDb to make compaction less painful, but with the prospect
of replacing the append-only format, my third goal would be to allow reusing
disk space.

## {{ anchor(text = "The API of Sediment" )}}

During the development of [Nebari][nebari], I had abstracted nearly every read
and write operation into something I called a "chunk". A chunk was the data
stored with 8 extra bytes: 4 bytes for a CRC32 to verify the data, and 4 bytes
for the length of the data. A chunk's ID was simply it's location in the file.

My goal of Sediment's design would be to focus on this primary concern: store
some arbitrary bytes and be given a unique identifier. That unique ID should be
able to retrieve or delete the data.

The only other feature Nebari needed is a way to store the "root" of the B+Tree
and retrieve it again when opening a database.

These considerations yielded this API:

```rust
// Store some data
let mut session = db.new_session();
let grain_id: GrainId = session.push(b"hello world")?;
session.commit()?;

// Retrieve the data
let data = db.get(grain_id)?.unwrap();
assert_eq!(data.as_bytes(), b"hello world");

// Archive the data (marks it for deletion)
let mut session = db.new_session();
session.archive(grain_id)?;
let archiving_commit: BatchId = session.commit()?;

// Free the archived data. Until this is complete, `grain_id` can still 
// retrieve the stored data.
db.checkpoint_to(archiving_commit)?;
```

A `GrainId` is an opaque 64-bit identifier for stored data, and a
`BatchId` is an identifier of a specific commit. Rather than offering an API to
free data immediately, Sediment uses an internal log that can enable
replication. Why did I design it this way?

There is one key distinction between Sediment and nearly every format I've
compared it against: Sediment does not use [write-ahead logging (WAL)][wal] or
journal file. Sediment only requires a single file for its storage, just like
Nebari did. My decision to do this was in an effort to minimize IO.

There are many numerous benefits for WAL, but one downside is that data written
to the database is often written multiple times. First, the new data is written
to the log. At some future point, the database engine will checkpoint the log,
and those changes will be applied to the primary data store. This is an extra
copy that can be avoided by Sediment, as there's no real benefit to keeping a
separate log for this format.

The log that Sediment uses simply tracks `GrainId` allocations and archivals.
When the log is checkpointed, each archived grain in the log entries that are
being removed will be freed. This simple approach provides enough information to
implement Sediment's consistency guarantees and could be used to replicate the
database.

## {{ anchor(text = "The Format of Sediment" )}}

The basic design behind Sediment is to divide the file into blocks of data that
are pre-allocated and can be relocated to defragment the file. Basins have a
list of Strata, and Strata have a list of "Grain Maps".

A `GrainId` is a 64-bit identifier that uses 8 bits to identify which Basin the
grain is located within. The next 8 bits identify the Stratum. The remaining 48
bits identify the index of the grain within the Stratum.

Each Stratum has configurable properties:

- Grain Length: How many bytes is each grain allocated?
- Pages per Map: Grain Maps are divided into pages. To support very large
  databases, each map can have multiple pages.
- Grain Map Count: How many grain maps are allocated?

One challenge this format has is keeping track of which regions of the file are
currently allocated for Basins, Strata, and Grains. These structures have been
designed to minimize the amount of data required to load enough state for the
database to be validated and the file's allocations to be fully tracked.

The commit log is stored separately from the Basins, Strata, and Grains. The
file's header points to the most recent page of commit log entries. If a
database is never checkpointed, there will be extra overhead on loading the
database to scan the commit log to understand which pages of the file have been
allocated and which are free. For most practical use cases, there will rarely be
more than a handful of pages of commit log entries.

Each header structure is stored twice within the file, and the "inactive" copy
is overwritten when saved. This copy-on-write behavior combined with the commit
log allows Sediment to verify that all writes performed during the last commit
are completely intact when reopening the database. If a CRC or data
inconsistency is found, the failed commit will be reverted before the database
is opened.

## {{ anchor(text = "The Architecture of Sediment" )}}

The two main types in understanding how Sediment works are the `Atlas` and the
`Committer`. The `Atlas` is responsible for keeping track of the in-memory state
of the disk structures. The `Committer` is responsible for persisting changes to
those structures to disk.

This current design does not allow multiple processes to share access to a
Sediment database, as there are in-memory contracts between the `Atlas` and
`Committer`. One example is how file allocations are tracked. The `Atlas` is
responsible for allocating disk space for each grain written. Conversely, the
`Committer` will update the `Atlas` when disk space can be reused.

The `Atlas` is the least optimized part of Sediment. The problem of determining
how to allocate a grain is not simple, and my initial implementation is a brute
force algorithm. It's been [on my radar][grain-allocation-refactor], and when
benchmarking Nebari's insert performance, grain allocation took nearly 70% of
CPU time. Additionally, the `Atlas` currently is behind a single mutex. This
might be good enough, but more complex concurrency management may be something
to investigate as well.

Why is grain allocation tricky? The storage format of grains allows allocating
multiple sequential grains as a single `GrainId`. For example, if 64-byte value
is being written, it could be stored in four 16-byte grains or it could be
stored in 64 1-byte grains. The only limit is that only 255 consecutive grains
can be allocated for a single value. This gives an incredible amount of
flexibility, but it also has led to many hours of theorycrafting the best
allocation algorithms.

To write data, a "session" is created. The session can be used to write data,
archive data, checkpoint the database, or update the embedded header. If the
session is dropped, all reserved grains are available to be reused immediately.

When the session is committed, the changes are batched with any other pending
changes from other sessions waiting to be committed. The session's thread checks
if another commit is already happening. If no other commit is happening, the
session's thread becomes the commit thread and commits the currently batched
changes. Otherwise, the thread will wait for a signal that its batch has been
committed or for another opportunity to become the commit thread.

This design was intended to allow significant read and write concurrency, and
I'm happy to report that the benchmarks show promising concurrency results.

## {{ anchor(text = "Benchmarking Sediment" )}}

Sediment isn't a general purpose database, nor is it a key-value store. I am not
aware of any other storage API to benchmark against that can offer a true
apples-to-apples comparison. Still, I wanted to have some meaningful
comparisons, so I initially benchmarked single-threaded insert performance
against SQLite. However, SQLite doesn't offer any multi-threaded concurrency
options that allow it to batch transactions. I identified [RocksDB][rocksdb] as
a good candidate, as it has a [great crate][rocksdb-crate] and supports batching
multithreaded transactions. Here are the results of the single-threaded benchmark:

|    Backend     |   avg   |   min   |   max   | stddev  |
|--------------|---------|---------|---------|---------|
|   rocksdb    | 2.591ms | 2.386ms | 2.797ms | 54.69us |
| sediment     | 2.765ms | 2.581ms | 6.020ms | 332.7us |
|    sqlite    | 4.347ms | 4.148ms | 6.944ms | 273.2us |

The [batch insert benchmark][benchmark] measures a variable number of threads
performing a series of batch commits. Here are the average times to commit each
batch insert operation with ACID guarantees on my AMD Ryzen 3700X with 8 cores
and a Samsung SSD 970 EVO Plus:

|  Backend   |   16 threads   |   32 threads   | 64 threads |
|----------|---------|---------|---------|
| rocksdb  | 6.044ms | 9.194ms | 17.83ms |
| sediment | 6.362ms | 7.281ms | 8.247ms |
|  sqlite  | 24.93ms | 46.60ms | 163.2ms |

I was very happy to see that the performance of Sediment scales very nicely even
with 64 threads concurrently writing to the database -- in spite of my naive
grain allocation algorithm. The next step was to see how much overhead Nebari
would add.

## {{ anchor(text = "Benchmarking Nebari on Sediment" )}}

The [Nebari benchmarks][nebari-benchmarks] are single-threaded and limited to
single trees, which made it much easier to quickly incorporate Sediment. There
are three main operations that are benchmarked: insert, get, and scan. Here are
the insert benchmark's results, which tests the speed of ACID-compliant
insertion of a blob of a specified size:

|  Backend   |   1 KB   |   1 MB   | 64 MB |
|----------|---------|---------|---------|
| nebari (sediment)  | 2.823 ms | 4.124 ms | 183 ms |
| nebari (v0.5.3) | 2.678 ms | 6.959 ms | 284.9 ms |
|  sqlite  | 4.182 ms | 7.2 ms | 159 ms |

For 1 KB inserts, the average performance is pretty similar. As mentioned
earlier, when profiling this benchmark, nearly 70% of the CPU time is spent in
grain allocation, which has been [on my refactor list since being
written][grain-allocation-refactor]. For larger inserts, however, Sediment is
much more efficient. How about the performance of retrieving a single row out of
various data set sizes (random ids):

|  Backend   |   1k rows   |   10k rows   | 1m rows |
|----------|---------|---------|---------|
| nebari (sediment)  | 749.6 ns | 908.7 ns | 9.067 us |
| nebari (v0.5.3) | 766.1 ns | 913.2 ns | 9.413 us |
|  sqlite  | 6.944 us | 7.471 us | 8.671 us |

As I expected, there isn't really much variation in performance. What about
scanning a range of keys (random IDs):

|  Backend   |   32 of 1k rows   |   100 of 10k rows   | 1k of 1m rows |
|----------|---------|---------|---------|
| nebari (sediment)  | 2.747 us | 72.35 us | 982.4 us |
| nebari (v0.5.3) | 2.684 us | 90.02 us | 1.156 ms |
|  sqlite  | 16.67 us | 44.75 us | 414.4 us |

This was one of the most surprising results to me. There is a ~15% speed
improvement on this [fairly unoptimized operation][refactor-scan] on the larger
data sets. My best guess as to why this operation is faster is that data stored
in Sediment is packed closer together, which makes it more likely that the
kernel page cache already has needed data in the cache.

Overall, these benchmarks have shown that replacing the append-only format with
Sediment has been a success so far.

## {{ anchor(text = "Next Steps" )}}

Nebari's multi-tree type, `Roots`, is not currently compatible with Sediment. To
finish realizing my goal of a single `fsync()` ACID transaction, I need to
update the `Roots` type to support multi-tree transactions within Sediment.
Until earlier this week, my vision was to refactor `Roots` to use a single file
for all trees by using a B+Tree of B+Trees internally.

However, with the results of the scan benchmark, I'm believe I'm seeing some
benefits of keeping "related data" together. I'm currently debating whether I
want to proceed with the single-file approach or whether I'm going to update the
current multi-file approach with a new transaction log that can be synchronized
in parallel with the affected trees.

I've started maintaining a [project on GitHub][bonsaidb-v0.5] tracking the
issues I want to complete before I release the next BonsaiDb update. There are
also a lot of areas left to explore:

- Sediment currently is optimized for 4 KB pages. ZFS, for example, has a 128 KB
  record size by default. Should Sediment offer customizable page sizes, or is
  the 4 KB page size fine even with filesystems that allocate in chunks larger
  than 4 KB? I've seen some mixed advice, so I need to do some extensive
  benchmarking on various filesystems to try to understand the impacts of the
  page size.
- The current pre-allocation strategy is fairly aggressive with a ~1:16 ratio.
  For example, let's say you're writing a 1 MB value to Sediment. The smallest
  grain size that can be used must be able to fit `ceil(1 MB / 255)`. This means
  that the smallest grain length that would be allocated is 4,096 bytes. Each
  grain map page contains 4,084 grains which means a minimum grain map
  allocation length would be 4 KB * 4,084, which is ~16 MB. If your database is
  10 GB, allocating 16 MB is a no-brainer, but if your database is only 1 MB, a
  16 MB allocation is very aggressive. I want to [add another
  way][oversized-blobs] to store large values that would not require such an
  agressive preallocation strategy.
- I've tried out using `tokio_uring` to add `io_uring` support on Linux, and my
  initial benchmarking results are that there is not much noticable difference
  compared against using `std::fs`. I need to spend some time profiling to
  understand if I'm doing something silly. I've read that using `O_DIRECT` with
  `io_uring` can be very beneficial, but the `tokio_uring` crate does not
  support specifying additional open flags yet.
- The CEO behind [Hibernating Rhinos][hibernating-rhinos], the company
  developing RavenDB, [wrote a response to my last post][ravendb-re]. They've
  had great success with direct IO even without `io_uring`. The basic idea is
  that by specifying a few flags, you can avoid the kernel's page caching if you
  always write full pages of data that are page-aligned. While I have not tested
  what sorts of gains Sediment can gain by using direct IO, the data structures
  in Sediment have been designed with this potential optimization in mind.
- This new format supports reusing data as well as moving the stored data
  around. The approaches for doing various defragmenting operations are
  currently just theoretical models in my head that need to be explored and
  implemented.

While this new format is much more complex than the current append-only format,
I've had enough promising results that I'm proceeding with this new format. My
next goal is to get the BonsaiDb Commerce Benchmark running atop Sediment --
that is the benchmark where the current format results in BonsaiDb to be roughly
2.5x slower than PostgreSQL. I'm optimistic that BonsaiDb can compete based on
these early results!

Assuming that the remaining development goes smoothly, I am currently hoping to
have BonsaiDb v0.5 out sometime in August with support for the new format. I am
committed to having a painless way to migrate data from the old format to the
new format, so all current users or people considering trying BonsaiDb need not
worry!

## {{ anchor(text = "Getting Started with BonsaiDb") }}

{{ gettingstarted() }}

[sediment]: https://github.com/khonsulabs/sediment
[nebari]: https://github.com/khonsulabs/nebari
[bonsaidb]: https://github.com/khonsulabs/bonsaidb
[nebari-benchmarks]: https://github.com/khonsulabs/nebari/tree/main/benchmarks
[couchdb]: https://couchdb.apache.org/
[wal]: https://en.wikipedia.org/wiki/Write-ahead_logging
[grain-allocation-refactor]: https://github.com/khonsulabs/sediment/issues/4
[rocksdb]: https://github.com/facebook/rocksdb
[rocksdb-crate]: https://crates.io/crates/rocksdb
[benchmark]: https://github.com/khonsulabs/sediment/blob/8b43f044a8daad85dbdf2218c2a7a7ced6bc9759/benchmarks/benches/inserts.rs
[refactor-scan]: https://github.com/khonsulabs/nebari/issues/11
[bonsaidb-v0.5]: https://github.com/orgs/khonsulabs/projects/10
[ravendb-re]: https://ayende.com/blog/197377-C/re-bonsaidb-performance-update-a-deep-dive-on-file-synchronization
[oversized-blobs]: https://github.com/khonsulabs/sediment/issues/13
[hibernating-rhinos]: https://hibernatingrhinos.com/
