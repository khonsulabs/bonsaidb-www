+++
title = "ACID Compliance: Syncing files fast"

[extra]
author = "Jonathan Johnson"
author_url = "https://github.com/ecton"
+++

{{ whatis() }}

## {{ anchor(text = "tl;dr: BonsaiDb is slower than previously reported")}}

The day after [the last post](/blog/april-2022-update), [@justinj][justinj]
reported that they traced one of the Nebari examples and did not see any `fsync`
syscalls being executed. This was an honest mistake of misunderstanding the term
"true sink" in [`std::io::Write`][write-docs]. It turns out `Write::flush()`'s
implementation for `std::io::File` is a no-op, as the "true sink" is the kernel,
not the disk.

I released [Nebari v0.5.3][nebari-release] the same day. I ran the Nebari
benchmark suite and... nothing changed. I ran the suite on GitHub Actions, no
change. I ran the suite on my dedicated VPS I use for a more stable benchmarking
environment than GitHub Actions... no change. I ran the suite on my Mac... huge
slowdown. I'll cover why further in this post, but my initial impression was
that I dodged a bullet somehow.

A few days later, I noticed the BonsaiDb Commerce Benchmark was running slowly.
I quickly realized it was due to the synchronization changes and [began an
entire rewrite of the view indexer and document storage][bonsaidb-refactor]. A
few days ago, I reached a stage where I could run the suite with my changes.
Excited to see if it was enough to catch back up to PostgreSQL, I ran it and...
it was a little faster but was still very slow.

The rest of this post explores everything I've learned since then. Since this a
summary, let's end with the **tl;dr: Reading data from BonsaiDb is still very
efficient, but due to mistakes in benchmarking, writes are quite slow for
workflows that insert or update a lot of data in a single collection. I am still
excited and motivated to build BonsaiDb, but I am currently uncertain whether I
will still write my own low-level database layer. All assumptions about
BonsaiDb's performance must be reset.**

## {{ anchor(text = "Why didn't my refactor help?")}}

The rest of that day and the next two days were spent profiling and trying to
understand the results. I would then try to test my assumptions in an isolated
test, and I wasn't able to make significant progress.

The following day as I was sipping coffee, I ran `df` to check my disk's free
space. I realized that each time I ran that command, `/tmp` was always listed as
a separate mountpoint. I use Manjaro Linux (based on Arch), and while I can
generally solve any problems I have with my computer, I never considered the
implications of `/tmp` listed.

Given that it's a separate mountpoint than my main filesystem, the next logical
question is: what filesystem does it use? The answer: [tmpfs][tmpfs], a
filesystem that acts like a RAM-disk and never persists your files except
through memory paging. `fsync` is essentially a no-op on such a filesystem. Many
of my benchmarks and tests used the excellent [`tempfile`][tempfile] crate.

Despite having worked with Linux off and on since the early 2000s, I never
noticed this detail. The temporary directory is not a different filesystem on
the Mac, which is one factor in why Nebari's benchmarks exhibited a major change
on the Mac while no change on Linux.

## {{ anchor(text = "Should I continue Nebari?" )}}

The realization that all of my testing of other database's performance was
severely flawed meant that I needed to recheck everything. There is a real
question: should I just ditch Nebari and use another database to back BonsaiDb?
Thinking about my motivations, I've never wanted to create "the fastest
database."

My pitch for BonsaiDb has always been about developer experience and being "good
enough" for "most people." I believe there are a lot of developers who waste a
lot of development cycles building apps that are more concerned about
high-availability than being able to scale to be the next Google. At that scale,
a one-size-fits-all solution is almost never the solution. If I can simplify
scaling from a test project to a highly-available solution and do it with
"reasonable" performance, I've met my goals for BonsaiDb.

The benefits of Nebari come down to it being tailor-fit to BonsaiDb's needs.
With my new approach to document and view storage that leverages Nebari's
ability to embed custom stats within its B+Tree, it meant I could build and
query map-reduce views in a very efficient manner. I have yet to see another
low-level database written in Rust that enables embedding custom information
inside of the B+Tree structure to enable these capabilities.

Because a tailor-fit solution is so attractive, I wanted to explore what it
might take to make Nebari faster.

## {{ anchor(text = "Why is Nebari slow?" )}}

To answer that question, I needed to fix my usage of `tempfile` in the Nebari
benchmark. After doing that, Nebari still competed with
[SQLite](https://sqlite.org/) on many benchmarks, but [Sled](https://sled.rs)
was reporting astonishing numbers. This led me to question whether Sled's
transactions are actually ACID-compliant when explicitly asking for them to be
flushed.

See, in my testing, I was able to determine that `fdatasync()` was unable to
return in less than 1 millisecond on my machine. Here's the output if the
transactional insert benchmark measuring the time it takes to do an
ACID-complaint insert of 1KB of data to a new key:

```sh
blobs-insert/nebari-versioned/1KiB
                        time:   [2.6591 ms 2.6920 ms 2.7348 ms]
                        thrpt:  [365.66 KiB/s 371.47 KiB/s 376.07 KiB/s]
blobs-insert/nebari/1KiB
                        time:   [2.6415 ms 2.6671 ms 2.7023 ms]
                        thrpt:  [370.05 KiB/s 374.93 KiB/s 378.57 KiB/s]
blobs-insert/sled/1KiB  time:   [38.438 us 38.894 us 39.376 us]                                    
                        thrpt:  [24.801 MiB/s 25.108 MiB/s 25.406 MiB/s]
blobs-insert/sqlite/1KiB
                        time:   [4.0630 ms 4.1192 ms 4.1913 ms]
                        thrpt:  [238.59 KiB/s 242.76 KiB/s 246.12 KiB/s]
```

As you can see, Nebari sits in the middle with 2.6ms, SQLite is the slowest with
4.11ms, and Sled reports an incredibly short 38.9μs. After a quick skim of
Sled's source, Sled isn't calling `fdatasync()` most of the time when you ask it
to flush its buffers. It's using a different API: `sync_file_range()`.

Does that mean Sled isn't persisting changes completely? No. Instead, this is an
example of how to misunderstand statistics. From the output above, you might be
tempted to infer that since the max single iteration time for Sled is 39.4μs.
However, Criterion uses sampling-based statistics, which means that it doesn't
look at individual iteration times but rather iteration times for a set number
of iterations.

By logging out each individual iteration time, I can see that there are
individual iterations that *do* take as long as an `fdatasync` call. Let's
simplify and just look at the performance of `fdatasync()` and
`sync_file_range()`. I created three tests to benchmark:

- `append`: Write to the end of the file and call `fdatasync`.
- `preappend`: When a write needs more space in the file, extend the file using
  `ftruncate` before writing. Call `fdatasync` after each write.
- `syncrange`: When a write needs more space, extend the file using `ftruncate`
  and calling `fdatasync` after the write. When a write does not need more
  space, call `sync_file_range` to persist the newly written data.

The output of [this benchmark][durable-writes-bench] is:

```sh
writes/append           time:   [2.6211 ms 2.6585 ms 2.7058 ms]
writes/preallocate      time:   [1.2467 ms 1.2675 ms 1.2923 ms]
writes/syncrange        time:   [190.59 us 193.03 us 195.83 us]
```

Our `syncrange` benchmark appears to not ever take longer than 195.8μs. But, we
know that it calls `fdatasync`, so what's happening? Let's open up Criterion's raw.csv report for `syncrange`:

| group | function | value | throughput_num | throughput_type | sample_measured_value | unit | iteration_count |
|-------|----------|-------|----------------|-----------------|-----------------------|------|-----------------|
| writes | syncrange |  |  |  | 2,929,333.0 | ns | 6 |
| writes | syncrange |  |  |  | 2,932,484.0 | ns | 12 |
| writes | syncrange |  |  |  | 5,701,286.0 | ns | 18 |
| writes | syncrange |  |  |  | 5,788,786.0 | ns | 24 |

Criterion isn't keeping track of every iteration, it keeps track of batches.
Because of this, when the final statistics are tallied, we only see what can be
gathered from these batch results. Let's run the same benchmark in my own
benchmarking harness:

|    Label    |   avg   |   min   |   max   | stddev  |  out%  |
|-------------|---------|---------|---------|---------|--------|
|   append    | 2.628ms | 2.095ms | 5.702ms | 147.0us | 0.004% |
| preallocate | 1.243ms | 612.3us | 4.042ms | 859.3us | 0.004% |
|  syncrange  | 189.1us | 12.83us | 2.847ms | 653.6us | 0.063% |

Using this harness, I can see now see results that make sense. The averages
match what we see from Criterion, but now our min and max show a wider range. We
can now see that even for the `syncrange` benchmark, some writes will take
2.8ms.

The takeaway is very important: using `sync_file_range`, Sled is able to make
the average write's time take 38.9μs, even though occasionally there will be
write operations that are longer due to the need for an `fdatasync`
when the file's size changes.

Given how much faster `sync_file_range()` is, is it safe to use to achieve
durable writes?

## {{ anchor(text = "What does sync_file_range do?" )}}

`sync_file_range()` has many modes of operation. At its core, it offers the
ability to ask the kernel to commit any dirty cached data to the filesystem. For
the purposes of this post, we are most interested in the mode of operation when
`SYNC_FILE_RANGE_WAIT_BEFORE`,  `SYNC_FILE_RANGE_WRITE`, and
`SYNC_FILE_RANGE_WAIT_AFTER` are passed.

With these flags, `sync_file_range()` is documented to wait for all dirty pages
within the range provided to be flushed to disk. However, the documentation for
this function advises that it is "extremely dangerous."

## {{ anchor(text = "What are 'durable writes'?" )}}

When writing data to a file, the operating system does not immediately write the
bits to the phsysical disk. This would be incredibly slow, even with modern
SSDs. Instead, operating systems will typically manage a cache of the underlying
storage and occasionally flush the updated information to the storage. This
allows writes to be fast and enables the operating system to attempt to schedule
and reorder operations for efficiency.

The problem with this approach occurs when the power suddenly is cut to the
machine. Imagine a user hits "Save" in their program, the program confirmed it
was saved, and suddenly the building's power dies. The program claimed it saved
the file, but upon rebooting, the file is missing or corrupt. How does that
happen? The file may have only been saved to the kernel's cache and never
written to the physical disk.

The solution is called flushing or syncing. Each operating system exposes one or
more functions to ensure all writes to a file have successfully been persisted
to the physical media:

- On Linux, it's [`fsync()`][fsync], [`fdatasync()`][fdatasync], and
  [`sync_file_range()`][syncfilerange].
- On Windows, it's [`FlushFileBuffers`][flushfilebuffers].
- On Mac/iOS, `fsync()` is available but does not provide the same guarantees as
  Linux. Instead, a call to `fcntl` with the `F_FULLFSYNC` option must be used
  to trigger a write to physical media.

Rust uses the correct APIs for each platform when calling `File::sync_all` or
`File::sync_data` to provide durable writes. The standard library does not
provide APIs to invoke the underlying APIs mentioned above. Thankfully, the
`libc` crate makes it easy to call the APIs we are interested in for this post.

## {{ anchor(text = "Linux: Is sync_file_range viable for durable writes?" )}}

The [man page for `sync_file_range()`][syncfilerange] includes this warning
(emphasis mine):

> This system call is **extremely dangerous and should not be used in portable
> programs**. None of these operations writes out the file's metadata.
> Therefore, unless the application is strictly performing overwrites of
> already-instantiated disk blocks, there are no guarantees that the data will
> be available after a crash. There is no user interface to know if a write is
> purely an overwrite. On filesystems using copy-on-write semantics (e.g.,
> *btrfs*) an overwrite of existing allocated blocks is impossible. **When
> writing into preallocated space, many filesystems also require calls into the
> block allocator, which this system call does not sync out to disk**. This
> system call does not flush disk write caches and thus does not provide any
> data integrity on systems with volatile disk write caches.

Sled is using it to achieve it's incredible speed, and [the author is
aware][sled-issue] of this warning. One of the commentors on the linked page
points out that RocksDB [has special code to disable using the API on
`zfs`][rocksdb-zfs]. Pebble, which is a Go port/spinoff of RocksDB, takes the
approach of [opting-in `ext4`][pebble-ext4]. Both RocksDB and Pebble seem to
still use `fsync`/`fdatasync` at various locations to ensure durability.

I decided to look at PostgreSQL's source as well. They use `sync_file_range()`'s
asynchronous mode to [hint to the OS][postgresql-syncfilerange] that the writes
need to be flushed, but they still issue `fsync` or `fdatasync` as needed.

I also looked to SQLite's source: no references. I could not find any relevant
discussion threads either.

I'm not an expert on any of these databases, so my skim of their codebases
should be taken with a grain of salt.

I tried finding any information about the reliability of `sync_file_range` for
durable overwrites on various filesystems, and I couldn't find anything except
these little bits already linked.

Lacking any definitive answer regarding whether it's able to provide durability
on any filesystems, I set out to test this myself.

### {{ anchor(text = "Testing sync_file_range's durability" )}}

I set up an Ubuntu 20.04 Server virtual machine running kernel
5.4.0-110-generic. While pondering how to best shut the machine down after the
call to `sync_file_range`, [@justinj][justinj] came to the rescue again by
pointing out that `/proc/sysrq-trigger` exists. He also shared [his blog
post][justinj-blog] where he performed similar tests against `fsync` while
exploring how to build a durable database log.

It turns out if you write `o` to `/proc/sysrq-trigger` on a Linux machine
(requires permissions), it will immediately power off. This greatly simplified
my testing setup.

I executed a VM using this command:

```sh
qemu-system-x86_64 \
    -drive file=ubuntu-server,format=qcow2 \
    -drive file=extra,format=qcow2 \
    -enable-kvm \
    -m 2G \
    -smp 1 \
    -device e1000,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22
```

In another terminal, I executed the various examples from the repository over
`ssh`. After each example executed, the virtual machine would automatically
reboot. By executing examples in a loop, I was able to run these commands for
extended periods of time. My results are:

| Filesystem | is `sync_file_range` durable? |
|------------|-------------------------------|
| btrfs      | No  |
| ext4       | Yes |
| xfs        | Yes |
| zfs        | No    |

### {{ anchor(text = "Safely extending a file's length with sync_file_range" )}}

My original testing of `sync_file_range` showed some failures, but after some
additional testing, I noticed it was only happening with either the first or
second test, but never on subsequent tests for filesystems I've labeled durable
above.

There are two examples that test `sync_file_range`:

- [`sync_file_range.rs`][sync-file-range-test]: When initializing the data file, zeroes are manually
  written to the file.
- [`sync_file_range_set_len.rs`][sync-file-range-set-len-test]: When
  initializing the data file, `File::set_len()` is called to extend the file,
  which is documented currently as:

  > If it is greater than the current file’s size, then the file will be
  > extended to size and have all of the intermediate data filled in with 0s.

Both examples use `File::sync_all()`, and both examples call `File::sync_all` on
the containing directories to sync the file length change.

On ext4 and xfs, my testing showed that I could reliably reproduce data loss on
the initial run in the `sync_file_range_set_len` example but not the
`sync_file_range` example. Subsequent runs were durable. Why is that?

Despite what the Rust documentation states, under the hood, `File::set_len` uses
`ftruncate`, which is documented as:

> If the file size is increased, the extended area shall appear as if it were
> zero-filled.

The distinction between "shall appear as if it were zero-filled" and "will be
extended to size and have all of the intermediate data filled in with 0s" is
subtle, but very important when considering the safety of `sync_file_range`. In
my earlier quote of `sync_file_range`'s warning, the second emphasis also seems
to relate to these findings.

From my testing, using `ftruncate` to fill pages with 0 will conflict with
`sync_file_range` on the first operation, but will likely succeed on future
tests on ext4 and xfs.

### {{ anchor(text = "Conclusions about sync_file_range" )}}

- `sync_file_range` is only safe to use on specific filesystems. Of the four I tested, `xfs`
  and `ext4` appear to be completely reliable in their implementations, and
  `zfs` and `btrfs` both are completely unreliable in their implementations.
- `sync_file_range` is only safe to use on fully initialized pages.
- `ftruncate` to extend a file does not fully initialize newly allocated pages
  with zeroes and may take shortcuts instead. This makes using `sync_file_range`
  on space allocated with `ftruncate` or similar operations unsafe to use.

## {{ anchor(text = "Mac OS/iOS: Does F_BARRIERFSYNC provide durable writes?" )}}

No. Apple's own [documentation makes this clear][mac-writes]:

> Some apps require a write barrier to ensure data persistence before subsequent
> operations can proceed. Most apps can use the fcntl(*:*:) F_BARRIERFSYNC for
> this.
>
> Only use F_FULLFSYNC when your app requires a strong expectation of data
> persistence.

Great, so `fnctl` with `F_FULLFSYNC` is what is used to instead of `fsync`.
Let's keep reading.

> Note that F_FULLFSYNC represents a best-effort guarantee that iOS writes data
> to the disk, but data can still be lost in the case of sudden power loss.

Apple really dropped the ball here. According to all available documentation I
can find: there is no way to run a truly ACID-compliant database on Mac OS. You
can get close, but a power loss could still result in a write being reported as
successfully persisted being gone after a power outage.

One interesting note is that SQLite uses `F_BARRIERFSYNC` by default for all of
its file synchronization on Mac/iOS. Optionally, you can use a `#pragma` to
enable usage of `F_FULLFSYNC`. Given the relative overhead of the two APIs in my
limited testing, I can understand their decision, but I'm not sure it's the best
default.

## {{ anchor(text = "Windows: Are there any APIs for partially syncing files?" )}}

No. Unless you are utilizing memory mapped files, the only API avialable on
Windows is [`FlushFileBuffers`][flushfilebuffers].

## {{ anchor(text = "What does all of this mean for BonsaiDb?" )}}

Astute readers may have noticed that the Nebari benchmarks claimed to be similar
in performance to SQLite even post-sync changes. This is true on many metrics,
but it's not an apples-to-apples comparison, and the difference is the primary
reason BonsaiDb slowed down.

SQLite has many approaches to persistence, but let's look at the journaled
version as it's fairly straightforward. When using a journal, SQLite creates a
file that contains information needed to undo the changes its about to make to
the database file. To ensure a consistent state that can be recovered after a
power outage at any point in this process, it must make at least two `fsync`
calls.

Nebari gets away with one `fsync` operation due to its append-only nature.
However, the moment you use the `Roots` type, there's one more `fsync` operation:
the transaction log. Thus, Nebari isn't actually faster than SQLite when a
transaction log is used, which is a requirement for multi-tree transactions.

This is further exacerbated by Nebari's Multi-Reader, Single-Writer model of
transactions. If two threads are trying to write to the same tree, one will
begin its transaction and finish it while the other has to wait patiently for
the lock on the tree to be released.

This two-step sync method combined with contention over a few collections is
what caused the Commerce Benchmark to grind to a halt after `fsync` was actually
doing real work. Individual worker threads would back up waiting for their turn
to modify a collection.

Nebari's architecture was designed in October, and I spent countless hours
profiling and testing its performance. Due to the aforementioned issues with my
methodology, so many of my performance assumptions were flat out wrong.

## {{ anchor(text = "What's next?" )}}

It's clear from these results that whatever solution is picked for BonsaiDb, it
needs to support a way to allow multiple transactions to proceed at the same
time. This is a much tougher problem to solve, and I'm uncertain I want to
tackle this problem myself.

For the longest time, I developed BonsaiDb with minimal "advertising." Imposter
syndrome prevented me from sharing it for most of 2021. Over the alpha period, I
finally started feeling confidence in its reliability. Now, I'm back to
questioning whether I should attempt a new version of Nebari.

On one hand, seeing that Nebari is still pretty fast after fixing this bug
should prove to me that I *can* write a fast database. On the other hand, I'm so
embarrassed I didn't notice these issues earlier, and it's demoralizing to think
of all the time spent building upon mistaken assumptions. Nebari will also need
to transition to a more complex architecture, which makes it lose some of the
appeal I had for it.

The only thing I can say with confidence right now is that I still firmly
believe in my vision of BonsaiDb, regardless of what storage layer powers it. I
will figure out my plans soon so that existing users aren't left in a lurch for
too long.

Lastly, I just want to say thank you to everyone who has supported me through
this journey. Despite the recent stress, BonsaiDb and Nebari have been fun and
rewarding projects to build.

[bonsaidb]: https://github.com/khonsulabs/bonsaidb
[justinj]: https://github.com/justinj
[justinj-blog]: http://justinjaffray.com/durability-and-redo-logging/
[nebari-release]: https://github.com/khonsulabs/nebari/releases/tag/v0.5.3
[write-docs]: https://doc.rust-lang.org/std/io/trait.Write.html
[bonsaidb-refactor]: https://github.com/khonsulabs/bonsaidb/pull/250
[tmpfs]: https://en.wikipedia.org/wiki/Tmpfs
[tempfile]: https://crates.io/crates/tempfile
[flushfilebuffers]: https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-flushfilebuffers
[syncfilerange]: https://man.archlinux.org/man/sync_file_range.2
[fsync]: https://man.archlinux.org/man/fsync.2
[fdatasync]: https://man.archlinux.org/man/fdatasync.2
[mac-writes]:
    https://developer.apple.com/documentation/xcode/reducing-disk-writes
[sled-issue]: https://github.com/spacejam/sled/issues/1351
[rocksdb-zfs]:
    https://github.com/facebook/rocksdb/blob/bed40e7266b55349ce9d2dce27aeb2055813a5fe/env/io_posix.cc#L160-L166
[pebble-ext4]:
    https://github.com/cockroachdb/pebble/blob/3355a02e7cec7a4cbf775421a89dfbe833818266/vfs/syncing_file_linux.go#L34-L36
[postgresql-syncfilerange]:
    https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/storage/file/fd.c;hb=e19272ef603bdb11a09e7f8500dc4e0fb4ec73de#l493
[sync-tests]: https://github.com/ecton/sync-tests
[durable-writes-bench]: https://github.com/ecton/sync-tests/blob/main/benches/durable-writes.rs
[sync-file-range-test]: https://github.com/ecton/sync-tests/blob/main/examples/sync_file_range.rs
[sync-file-range-set-len-test]: https://github.com/ecton/sync-tests/blob/main/examples/sync_file_range_set_len.rs
