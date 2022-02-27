+++
title = "BonsaiDb February update: Supporting and Optimizing BonsaiDb"

[extra]
author = "Ecton"
author_url = "https://github.com/ecton"
+++

> **What is BonsaiDb?**
>
> [BonsaiDb][bonsaidb] is a new database aiming to be the most developer-friendly
> Rust database. BonsaiDb has a unique feature set geared at solving many common
> data problems. We have a page dedicated to answering the question: [What is
> BonsaiDb?](https://bonsaidb.io/about).

It's the last weekend of the month, and I want to make it a habit at the end of
each month to reflect on the previous month as well as talk about what I'm
currently focused on or what is coming next.

## {{ anchor(text = "Status of the Alpha") }}

I released our initial alpha on February 5. I am ecstatic with the warm
reception. [Our Discord server][discord] has grown with several active users who
have been running BonsaiDb through its paces. Several bugs have been discovered
and fixed, with these releases happening in February:

- [v0.1.0](https://github.com/khonsulabs/bonsaidb/releases/tag/v0.1.0): Initial alpha release.
- [v0.1.1][v0.1.1]: [Derive macros](#Derive%20Macros) for common traits, many documentation
  improvements, and memory-only databases for testing purposes.
- [v0.1.2](https://github.com/khonsulabs/bonsaidb/releases/tag/v0.1.2): Worked
  around an undocumented panic within tokio::sleep on Windows that a user
  experienced. Upstream patch merged as well.
- [v0.2.0](https://github.com/khonsulabs/bonsaidb/releases/tag/v0.2.0): [Custom
  primary keys](./bonsaidb-v0-2-0#Custom%20Primary%20Keys), [LZ4
  compression][compression], and other minor improvements.

In addition to these releases, the underlying storage layer ([Nebari][nebari])
has been updated as well with several new features, bug fixes, and performance
improvements. The primary performance improvements that have been released have
centered around better usage of Rust's type system to avoid extra allocations.

## {{ anchor(text = "Derive Macros") }}

[@ModProg][modprog] noticed that I had open issues for adding derive macros for
some common traits. He became the second contributor to the BonsaiDb repository
when these macros were added in [v0.1.1][v0.1.1].

I have an important design philosophy with BonsaiDb: Everything should be as
ergonomic as possible without macros. Macros should be used to improve an
already-well-designed API. I was confortable writing
schemas by hand, but I recognized how often I copy and pasted boilerplate code
and then tweaked small portions -- a good sign that a macro would help!

The way these macros are written is using a new crate [@ModProg][modprog] wrote:
[attribute-derive](https://github.com/ModProg/attribute-derive). I've made a few
updates to the macros since his pull request, and it's been a wonderful
experience. If you are looking into writing an attribute proc macro, you might
want to consider using this crate to make simplify parsing and support more
Rust-y syntaxes.

## {{ anchor(text = "Supporting BonsaiDb") }}

I've tried to be very responsive to all questions, feedback, and issues. It's
fun seeing BonsaiDb evaluated in new ways. As I recalled in [the LZ4 Compression
feature introduction][compression], I introduced that feature the day after a
potential user ran into space issues when evaluating BonsaiDb against their
current Sled code.

[@ModProg][modprog] one morning tried running BonsaiDb on a Raspberry Pi, which
led to us diagnosing [a bug/edge case that's been fixed in the `sysinfo`
crate](https://github.com/GuillaumeGomez/sysinfo/issues/700). It was a fun
morning debugging BonsaiDb, an async codebase, over a Discord screenshare
showing an ssh session into the Raspberry Pi running GDB!

A few days ago, the same user that was testing a somewhat large dataset reported
an issue. It turned out to be [a long-standing
bug](https://github.com/khonsulabs/nebari/commit/125234cb3928ad4cd0c087d1afdd1d25ecd5710e)
in Nebari's transaction log reading code. The unit tests were inadequate, and
the only reason it was noticed was due to the LZ4 decompression failing once a
transaction log entry was large enough to be stored compressed across multiple
pages. The associated data isn't currently being used outside of the
`list_executed_transactions()` function, but eventually is planned to be a core
part of replication. A `cargo update` is all that is needed for BonsaiDb to
adopt the updated version of Nebari.

## {{ anchor(text = "Testing BonsaiDb with a large dataset") }}

Despite the bug that was just found, I still felt fairly confident in BonsaiDb's
stability. Yet, it showed I hadn't really tested a big dataset. With a user
trying to use BonsaiDb with a large dataset, I felt like I should do my own
testing to feel as confident in BonsaiDb's abilities as I can be.

After looking through a few locations, I stumbled upon the [Open Library Data
Dumps](https://openlibrary.org/developers/dumps). The data set I'm going to talk
about in this post contains these entities:

| Collection | Records    |
|------------|------------|
| Authors    |  9,145,605 |
| Editions   | 33,102,390 |
| Works      | 24,010,896 |

The uncompressed file that this data is loaded from is 51,447mb. It contains
additional data from the types listed, so direct size comparisons aren't able to
be done post-import.

Writing an importer was a fun project. Since the data being imported is too
large to read into RAM before processing, I started with [a streaming TSV
parser](https://github.com/khonsulabs/bonsaidb/blob/5216db14f621233c0a5f7f8ba2ac62c7508b3bd5/examples/open-library/examples/open-library.rs#L440-L481)
that uses [flume][flume] to send parsed records to another task. This uses a
bounded channel which will park the TSV parser thread if it outpaces the code
that saves the records to the disk.

The data importing logic is fairly straightforward. I have one method that
receives records from the TSV parser and [creates batched
transactions](https://github.com/khonsulabs/bonsaidb/blob/5216db14f621233c0a5f7f8ba2ac62c7508b3bd5/examples/open-library/examples/open-library.rs#L545-L565).

Importing the data with LZ4 compression enabled took 60m56s, and with
compression disabled it took 64m12s. The resulting file sizes are interesting:

| Collection | LZ4 Uncompacted | LZ4 Compacted | Uncompacted | Compacted |
|------------|-----------------|---------------|-------------|-----------|
| Authors    |       13,316mb |        3,675mb |    23,508mb |   4,159mb |
| Editions   |       74,459mb |       33,155mb |   118,330mb |  38,372mb |
| Works      |       40,443mb |       12,082mb |    71,601mb |  13,565mb |

BonsaiDb uses [Nebari][nebari], which uses an append-only file format for its
database. As the internal trees are built, old nodes are left behind on disk.
The only way to reclaim the disk space currently is to compact the database -- a
process which rewrites all the active data to a fresh file and swaps the files
atomically once fully synchronized. Compacting the compressed database took
6m55s, and compacting the uncompressed database took 7m20s.

After testing importing and overwriting the database several times, my
confidence level in BonsaiDb grew. I built a simple CLI that allows querying
authors, editions, and works by their unique ID. Everything was working great.

### {{ anchor(text = "Querying large datasets with views") }}

I wanted to add support to show all of the books (works) that an author wrote.
In the data coming from OpenLibrary, the `Work` type has a list of author roles.
This is the
[`View`](https://dev.bonsaidb.io/release/guide/about/concepts/view.html)
definition that enables this query:

```rust
#[derive(View, Debug, Clone)]
#[view(name = "by-author", collection = Work, key = String, value = u32)]
struct WorksByAuthor;

impl CollectionViewSchema for WorksByAuthor {
    type View = Self;

    fn map(
        &self,
        document: CollectionDocument<<Self::View as View>::Collection>,
    ) -> ViewMapResult<Self::View> {
        document
            .contents
            .authors
            .into_iter()
            .filter_map(|role| role.author)
            .map(|author| {
                document
                    .header
                    .emit_key_and_value(author.into_key().replace("/a/", "/authors/"), 1)
            })
            .collect()
    }
}
```

The data has some sanitization issues, including that keys sometimes are
shortend from `/authors/UNIQUEID` to `/a/UNIQUEID`. Because importing the data
takes so long, I chose to clean up the data post-import rather than force myself
to re-run the import with some code to clean it up while importing.

The entire function to display an author in the CLI is:

```rust
async fn summarize(&self, database: &Database) -> anyhow::Result<()> {
    if let Some(name) = &self.name {
        println!("Name: {}", name);
    }
    if let Some(bio) = &self.bio {
        println!("Biography:\n{}", bio.value())
    }
    let works = database
        .view::<WorksByAuthor>()
        .with_key(self.key.clone())
        .query_with_collection_docs()
        .await?;
    if !works.is_empty() {
        println!("Works:");
        for work in works.documents.values() {
            if let Some(title) = &work.contents.title {
                println!("{}: {}", work.contents.key, title)
            }
        }
    }

    Ok(())
}
```

While this was easy to implement, I knew that this was going to be a painful
process to run. BonsaiDb's views are lazily evaluated (with the exception of
unique views). The first time the view is accessed, the view will need to have
it's `map()` function invoked for each document stored in the collection. For
the Works collection, that means processing 24 million rows.

The initial query took over an hour. Subsequent queries are fast: the `time
open-library author OL1394865A` command completes in less than 400ms once the
view is indexed.

I really wanted to be able to turn this example into a benchmarking suite, and
the only way to do that without driving myself crazy was to start optimizing.

### {{ anchor(text = "Optimizing View Mapping") }}

The initial implementation of the view mapper was aimed at being correct, not
necessarily fast. It was originally designed to operate within
[Sled](https://sled.rs/), which allowed non-transactional reads and writes. The
thought was that I could simply parallelize some loops and magically the view
mapping system would be faster because Sled would handle ensuring the data was
written to disk correctly with multi-threaded writes.

After switching to [Nebari][nebari], the view mapper always runs in a
transactional context. Each view mapping operation created its own transaction
-- the slowest way to write to the database. The biggest priority was going to
be to refactor the view mapper to batch the operations into larger transactions.

My [initial refactoring did just
that](https://github.com/khonsulabs/bonsaidb/pull/213/commits/b40607f753f1ab1808eb25a559e39fa0f6080541)
and lowered the execution time significantly (~70%), but it also introduced a subtle
bug. As a result, not all mappings were being saved, but I
didn't notice the issue until much later in the refactoring process.

One issue I was running into is that Nebari's `TransactionTree` required an
exclusive borrow on the `ExecutingTransaction`. This prevented being able to read
documents in one thread while writing view entries in another thread. I
[refactored Nebari's transaction handling
code](https://github.com/khonsulabs/nebari/commit/c195be84412b532dd1802379676cada106f6c62f)
to allow this sort of parallelization.

After [adopting the new
functionality](https://github.com/khonsulabs/bonsaidb/pull/213/commits/5216db14f621233c0a5f7f8ba2ac62c7508b3bd5),
here are the view mapping times on the compressed dataset:

| Collection |     Size | View           | Unoptimized (LZ4) | Optimized (LZ4) | Optimized | Uncompacted | Compacted |
|------------|----------|----------------|-------------------|-----------------|-----------|-------------|-----------|
| Works      | 12,082mb | WorksByAuthor  |          1h02m03s |          10m16s |    10m13s |    16,241mb |   1,863mb |
| Editions   | 33,155mb | EditionsByWork |            26m52s |          12m23s |    12m48s |    17,507mb |   3,396mb |

While preparing this post, I've re-run the pre-optimized code and optimized code
to verify my notes on timings. When initially starting work on optimization, I
hadn't implemented `EditionsByWork`. I've verified the output of both
unoptimized versions and optimized versions match a random set of entries that
I've compared against their website. To clarify some of the numbers: all sizes
listed above are the compressed sizes. Adding all sizes made the table too large
to be easily understood.

At this stage, I no longer have any low hanging fruit from an algorithm
perspective. The next step in optimization would be to start profiling, but I've
reached my initial goals, so I'm going to be focusing on wrapping up this [pull
request](https://github.com/khonsulabs/bonsaidb/pull/213) by finishing this
example. I would enjoy turning this into a benchmark suite, but I'm also not
excited at the prospects of how long such a suite would take to debug fully.

## {{ anchor(text = "Next steps for BonsaiDb") }}

I'm going to continue prioritizing any stability or performance issues as they
are reported. There have been a few repeated requests that I'm going to be
working on soon: "best practices" overview, internal archicture documentation,
and a high-level roadmap.

Several people have expressed interest in learning more about BonsaiDb's
internals and roadmap with a goal of potentially contributing. I will be working
on those requests soon. In the meantime, I want to extend my offer to anyone
reading this: if you have any questions about any part of BonsaiDb, don't
hesitate to ask! Regardless of your experience with Rust, if you're excited at
the idea of helping build BonsaiDb, I will be happy to help you get up to speed.

Aside from improved documentation and project planning, I was working on
building a [persistent job
queue](https://github.com/khonsulabs/bonsaidb/pull/211), which I will be
returning to and hopefully ship in March. I'm building this feature as a
standalone crate that will allow users to create independent job queues with
different job distribution strategies, and the long-term goal is for this crate
to be the first "plugin" for BonsaiDb.

## {{ anchor(text = "Getting Started") }}

Our [homepage](https://bonsaidb.io/) has basic setup instructions and a list of
examples. We have started writing a [user's
guide](https://dev.bonsaidb.io/v0.2.0/guide/), and we have tried to write [good
documentation](https://docs.rs/bonsaidb/).

We would love to hear from you if you have questions or feedback. We have
[community Discourse forums](https://community.khonsulabs.com/) and a [Discord
server](https://discord.khonsulabs.com), but also welcome anyone to [open an
issue](https://github.com/khonsulabs/bonsaidb/issues/new) with any questions or
feedback.

We dream big with [BonsaiDb][bonsaidb], and we believe that it can simplify
writing and deploying complex, data-driven applications in Rust. We would love
[additional
contributors](https://github.com/khonsulabs/bonsaidb/labels/good%20first%20issue)
who have similar passions and ambitions.

Lastly, if you build something with one of our libraries, we would love to hear
about it. Nothing makes us happier than hearing people are building things with
our crates!

[discord]: https://discord.khonsulabs.com/
[bonsaidb]: https://github.com/khonsulabs/bonsaidb
[nebari]: https://github.com/khonsulabs/nebari
[v0.1.1]: https://github.com/khonsulabs/bonsaidb/releases/tag/v0.1.1
[compression]: /blog/bonsaidb-v0-2-0#LZ4%20Compression
[modprog]: https://github.com/ModProg/
[flume]: https://github.com/zesterer/flume
