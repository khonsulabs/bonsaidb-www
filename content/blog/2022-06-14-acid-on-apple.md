+++
title = "SQLite on macOS: Not ACID compliant with the bundled version"

[extra]
author = "Jonathan Johnson"
author_url = "https://github.com/ecton"
summary = "In an effort to understand a benchmark I wrote, I tested how Apple altered SQLite's implementation in macOS."
+++

I'm building [a database](/about), and I consider SQLite a "gold standard" to
compare my database against. While benchmarking new code recently, I noticed
Apple's bundled version of SQLite is not ACID compliant.

I do not consider myself an expert on these topics. If there are any errors in
my analysis, please reach out to me, and I will correct them immediately. I'm
learning by doing, and as evidenced by [my last post on this
topic](/blog/durable-writes), I have made my own fair share of mistakes in
trying to implement a fast, ACID-compliant database.

## {{ anchor(text = "Confusion about SQLite and F_BARRIERFSYNC" )}}

On February 17, 2022, [Scott Perry wrote this in a conversation on
Twitter:][numist-tweet]

> There's a third sync operation that lets you have your performance and write
> ordering too: F_BARRIERFSYNC. SQLite already uses it on Darwin, and it's part
> of the best practices guide for I/O reduction.
> <https://developer.apple.com/documentation/xcode/reducing-disk-writes>

Some people (myself included) interpretted the statement "SQLite already uses it
on Darwin" to mean that it's the default behavior. My post will show that this
is not the case. By default, the bundled version of SQLite distributed in macOS
12.4 (21F79) relies on `fsync()` for synchronization.

From my investigation, Apple's version of SQLite instead replaces `PRAGMA
fullfsync = on`'s implementation to use `F_BARRIERFSYNC`.

SQLite users who are expecting `PRAGMA fullfsync` to provide durability
guarantees in the event of power failures or kernel panics can [override xSync
via a custom VFS][numist-tweet2] or build SQLite from source.

To understand why, let's review how to ensure persistent writes on Apple's
operating systems.

## {{ anchor(text = "Recap: How to guarantee data is saved to disk on Mac/iOS?" )}}

There are two APIs we need to cover: `fsync()` and `fcntl()`. On Linux,
`fsync()` is the system call that is tells the kernel to fully synchronize its
file state with the disk. It is debatable whether the original POSIX
specification intends for this level of durability guarantees of `fsync()`, but
on Linux it tries its best to guarantee all bits changed have been synchronized
to the disk including issuing a flush of any affected volatile write caches.

However, on macOS, the man page for `fsync()` reads:

> Note that while fsync() will flush all data from the host to the drive (i.e.
> the "permanent storage device"), the drive itself may not physically write the
> data to the platters for quite some time and it may be written in an
> out-of-order sequence.
>
> Specifically, if the drive loses power or the OS crashes, the application may
> find that only some or none of their data was written.  The disk drive may
> also re-order the data so that later writes may be present, while earlier
> writes are not.
>
> This is not a theoretical edge case.  This scenario is easily reproduced with
> real world workloads and drive power failures.
>
> For applications that require tighter guarantees about the integrity of their
> data, Mac OS X provides the F_FULLFSYNC fcntl.  The F_FULLFSYNC fcntl asks the
> drive to flush all buffered data to permanent storage.  Applications, such as
> databases, that require a strict ordering of writes should use F_FULLFSYNC to
> ensure that their data is written in the order they expect.  Please see
> fcntl(2) for more detail.

Apple's documentation clearly states that for any guarantees about data loss due
to power loss or kernel panic, you must use the `fcntl()` API with the
`F_FULLFSYNC` command.

Back in February of this year, this topic circulated fairly widely, and [this
post from Michael Tsai][mjtsai] has a summary of the findings. In short, it was
noted that `F_FULLFSYNC` is incredibly slow in its current implementation. It
was noted that Apple points users to another `fcntl()` command in its ["Reducing
Disk Writes"][reducing-writes] article:

> Some apps require a write barrier to ensure data persistence before subsequent
> operations can proceed. Most apps can use the fcntl(_:_:) F_BARRIERFSYNC for
> this.
>
> Only use F_FULLFSYNC when your app requires a strong expectation of data
> persistence. Note that F_FULLFSYNC represents a best-effort guarantee that iOS
> writes data to the disk, but data can still be lost in the case of sudden
> power loss.

`F_BARRIERFSYNC` issues an IO barrier such that all subsequent IO operations
must wait for all current writes to succeed. The `fcntl()` call returns after
issuing the barrier, but before the data is synchronized. This is why using
`F_BARRIERFSYNC` doesn't fulfill the durability requirement of ACID: the changes
are confirmed before the data is fully synchronized.

I should note that while `fcntl()` is an API that is available on Linux,
`F_FULLFSYNC` and `F_BARRIERFSYNC` are specific to Apple OSes. Linux has no need
for these options as `fsync()` provides the guarantees needed.

## {{ anchor(text = "Does SQLite use F_BARRIERFSYNC on Apple OSes?" )}}

When starting my new low-level storage layer ([Sediment][sediment]), I added
support to optionally use `F_BARRIERFSYNC` instead of `F_FULLFSYNC` on macOS.
By default, `F_FULLSYNC` would still be used as I wanted the user to explicitly
opt-out of ACID if they needed the extra performance on Apple hardware. This was
based on the idea that SQLite was using this same approach to achieve its very
fast speed on macOS.

Yesterday, I created a simple benchmark to see where Sediment's performance was
currently at. I'm not ready to share numbers, and that's not the point of this
post. The summary, however, is that Sediment was faster than SQLite on Linux,
but slower than SQLite on my M1 Macbook Air.

That puzzled me, because if both SQLite and Sediment are using the same
synchronization primitives, how could the performance difference be inverted
between by switching operating systems? I decided to investigate how SQLite
utilized `F_BARRIERFSYNC`.

My first stop was the documentation. SQLite has a pragma to [enable
`F_FULLFSYNC`][pragma-fullfsync], but I could not find any documentation talking
about `F_BARRIERFSYNC`. The documentation for `PRAGMA fullfsync` states that the
default value is off.

My next stop was the SQLite source code: `full_fsync()` is defined in
[os_unix.c][sqlite-source]. Its responsibility is to perform a full fsync based
on the available and configured options. This section is what is relevant for
Apple OSes:

```c
#elif HAVE_FULLFSYNC
  if( fullSync ){
    rc = osFcntl(fd, F_FULLFSYNC, 0);
  }else{
    rc = 1;
  }
  /* If the FULLFSYNC failed, fall back to attempting an fsync().
  ** It shouldn't be possible for fullfsync to fail on the local 
  ** file system (on OSX), so failure indicates that FULLFSYNC
  ** isn't supported for this file system. So, attempt an fsync 
  ** and (for now) ignore the overhead of a superfluous fcntl call.  
  ** It'd be better to detect fullfsync support once and avoid 
  ** the fcntl call every time sync is called.
  */
  if( rc ) rc = fsync(fd);

#elif defined(__APPLE__)
  /* fdatasync() on HFS+ doesn't yet flush the file size if it changed correctly
  ** so currently we default to the macro that redefines fdatasync to fsync
  */
  rc = fsync(fd);
```

The SQLite source code shows the implementation for calling `fcntl()` with
`F_FULLFSYNC`, but has no mention of `F_BARRIERFSYNC`.

There's one last thing to check: maybe Apple ships a custom build of SQLite that
utilizes `F_BARRIERFSYNC`. The best way to verify is to use dtrace to log out
the system calls the process makes.

I disabled System Integrity Protection so that I could trace the `sqlite3`
executable that ships with macOS 12.4 (21F79):

```sh
~ % sudo dtruss -t fcntl sqlite3 test.sqlite
SYSCALL(args)    = return
SQLite version 3.37.0 2021-12-09 01:34:53
Enter ".help" for usage hints.
fcntl(0x3, 0x5F, 0x1)   = 0 0
fcntl(0x3, 0x3F, 0x6BDBD9C0)   = 3 0
fcntl(0x4, 0x32, 0x16BDBDDA8)   = 0 0

sqlite> insert into test (a) values (1);
fcntl(0x3, 0x5A, 0x16BDBD0B8)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBD0B8)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBD0B8)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBCBA8)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDE98)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDE98)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDE98)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDF08)   = 0 0
fcntl(0x4, 0x5F, 0x1)   = 0 0
fcntl(0x4, 0x3F, 0x1)   = 3 0
fcntl(0x5, 0x5F, 0x1)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDEE8)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDEE8)   = 0 0
fcntl(0x5, 0x5F, 0x1)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDE88)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDE88)   = 0 0
fcntl(0x3, 0x5A, 0x16BDBDEB8)   = 0 0
```

This log shows all of the `fcntl()` calls issued by SQLite to perform the
`insert into test (a) values (1);` statement. The second argument is the
command. We can see SQLite is using commands 0x3F, 0x5A, and 0x5F. In decimal,
those are 63, 90, and 95 respectively. Looking in `fcntl.h`, we see these values:

```c
#define F_FULLFSYNC             51      /* fsync + ask the drive to flush to the media */
#define F_GETPROTECTIONCLASS    63      /* Get the protection class of a file from the EA, returns int */
#define F_BARRIERFSYNC          85      /* fsync + issue barrier to drive */
```

The commands for 90 and 95 [are private][private-fcntl]:

```c
#define F_OFD_SETLK             90      /* Acquire or release open file description lock */
#define F_SETCONFINED           95      /* "confine" OFD to process */
```

We did not see any `fcntl()` calls with the command argument being 85 (0x55).
Let's try enabling `PRAGMA fullfsync`:

```sh
sqlite> pragma fullfsync=on;
sqlite> insert into test (a) values (1);
fcntl(0x3, 0x5A, 0x16DD9DEF8)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DEF8)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DEF8)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DF68)   = 0 0
fcntl(0x4, 0x5F, 0x1)   = 0 0
fcntl(0x4, 0x3F, 0x1)   = 3 0
fcntl(0x3, 0x5A, 0x16DD9DF48)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DF48)   = 0 0
fcntl(0x4, 0x55, 0x0)   = 0 0
fcntl(0x5, 0x5F, 0x1)   = 0 0
fcntl(0x4, 0x55, 0x0)   = 0 0
fcntl(0x3, 0x55, 0x0)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DEE8)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DEE8)   = 0 0
fcntl(0x3, 0x5A, 0x16DD9DF18)   = 0 0
```

As expected, we have a new `fcntl()` command: 0x55. Unexpectedly, however,
instead of enabling `F_FULLFSYNC` (0x33) as we would expect from reading the
publicly available SQLite code, we see 0x55 instead which is `F_BARRIERFSYNC`.

To summarize, Apple's SQLite doesn't use `F_BARRIERFSYNC` or `F_FULLFSYNC` by
default, and it replaces `fcntl(.., F_FULLFSYNC, ..)` with `fcntl(..,
F_BARRIERFSYNC, ..)` when `PRAGMA fullfsync` is enabled.

This behavior was [confirmed by Scott
Perry](https://twitter.com/numist/status/1536830264214638593) as I was editing
this post.

## {{ anchor(text = "How important is this?" )}}

For most consumer applications, `F_BARRIERFSYNC` will be enough to provide
reasonable durability with the benefit of performing much more quickly. However,
there are some situations where true ACID compliance is desired. Many (but not
all) of those situations involve server software.

With Apple no longer shipping server hardware and the performance of
`F_FULLFSYNC` [on Apple's drives][marcan_42], it's hard to fault Apple for
making the decision to use `F_BARRIERFSYNC` in their version of SQLite. I wish
they would have opted to do it in a different way, such as a new pragma or
changing the default `fsync()` behavior instead of replacing `PRAMGA fullfsync`.

It's very confusing when a feature that's [documented to be specific to
macOS][pragma-fullfsync] doesn't behave as documented on macOS. As it stands, if
a developer wants the documented behavior, the easiest way probably is to build
SQLite from source.

I did not test any of these findings on iOS -- it's been years since I have
tried doing any tracing on a device. I suspect Apple doesn't maintain separate
versions of SQLite for iOS and macOS, but because their version of SQLite is
closed source, we cannot verify easily.

Regardless of whether Apple changes how SQLite synchronizes in the future, I
encourage Apple to publish their updates to SQLite alongside their other open
source repositories. I can't imagine the changes made to SQLite would be
considered proprietary, and the ability to understand what differs between
SQLite's source code and the shipping version in Apple's operating systems is
important in understanding what guarantees SQLite provides on Apple's hardware.

[reducing-writes]: https://developer.apple.com/documentation/xcode/reducing-disk-writes
[mjtsai]: https://mjtsai.com/blog/2022/02/17/apple-ssd-benchmarks-and-f_fullsync/
[sediment]: https://github.com/khonsulabs/sediment
[pragma-fullfsync]: http://www3.sqlite.org/pragma.html#pragma_fullfsync
[sqlite-source]: https://sqlite.org/src/file?name=src/os_unix.c&ci=b1be2259
[private-fcntl]: https://github.com/apple/darwin-xnu/blob/2ff845c2e033bd0ff64b5b6aa6063a1f8f65aa32/bsd/sys/fcntl.h#L360-L370
[sqlite-wal]: http://www3.sqlite.org/wal.html
[numist-tweet]: https://twitter.com/numist/status/1494392674014531593
[numist-tweet2]: https://twitter.com/numist/status/1536859148897226753
[marcan_42]: https://news.ycombinator.com/item?id=30371857
