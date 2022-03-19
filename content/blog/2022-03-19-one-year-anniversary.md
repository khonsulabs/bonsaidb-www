+++
title = "A Year of BonsaiDb: A retrospective and looking to the future"

[extra]
summary = "BonsaiDb's primary contributor reflects on the first year of development and what the future holds for BonsaiDb."
author = "Jonathan Johnson"
author_url = "https://github.com/ecton"
+++

{{ whatis() }}

Today marks the one year anniversary of the [initial commit][initial-commit] to
[BonsaiDb][bonsaidb]. This project is largely a written by [me (@Ecton)][ecton],
although I'm very thankful to have been accompanied along the way by
[@daxpedda][daxpedda]. He is responsible for the [QUIC networking
layer][fabruic], our currently removed OPAQUE-KE support, and above all,
countless long discussions and debates that not only led to starting this
project, but also to the easy-to-use API designs we have today.

In the last few months, we've also had two additional contributors:
[@ModProg][modprog] and [@vbmade2000][vbmade2000]. I'm grateful for their
additions to the project, and I hope this year we see even more contributors as
BonsaiDb gains momentum.

## {{ anchor(text = "Beginning a real bonsai cluster") }}

The name BonsaiDb was suggested by [@daxpedda][daxpedda] after countless times
where we stumbled trying to pronounce the project's former name: PliantDb. When
I [announced the new name][name-change], I researched trying to grow a bonsai
tree in the Las Vegas climate. My initial findings were that many of the iconic
types are difficult to grow here, and I ended up shelving the idea.

As March was approaching, I knew I wanted to commemorate the anniversary of the
first commit. I researched again and found two species that were great
candidates for being able to be grown mostly indoors. After some deliberation, I
decided to start with a Golden Gate (Tiger Bark) Ficus:

{{ blockimage(alt = "One Year Anniversary Golden Gate Ficus", src ="/images/one-year-anniversary-ficus.jpg", width=400) }}

After giving the tree another week or two of adjustment to its new home, I'll be
pruning it for the first time and attempting to grow a second tree from its
cuttings. Within the limits of being able to provide good living conditions for
each tree, I hope to expand the "cluster" each year and will share progress
pictures along the way!

### {{ anchor(text = "The first year: Inspiration") }}

The inspiration for [BonsaiDb][bonsaidb] came from weeks and weeks of
discussions between [myself][ecton] and [@daxpedda][daxpedda]. We were wanting
to build an cluster architecture for an MMO that:

- Was easy to deploy and maintain
- Was affodable
- Could scale horizontally
- Could subdivide players by "location" within the cluster:
  - Each location could have database/cache/cpu resources allocated dynamically
  - Each location would be Highly Available
- Used as many Rust-written solutions as possible

At the start of our discussions, I was dead-set on using PostgreSQL and Redis
for our database/caching needs. However both PostgreSQL and Redis aren't easy to
deploy and maintain in a highly available cluster, and if I chose managed
solutions, they definitely weren't affordable.

When starting my last company, I chose [CouchDB][couchdb] for various reasons
that aren't worth diving into. Through the ten years of growing that company, I
became intimately familiar with the simple yet powerful design of CouchDB.
Several times I pondered rewriting CouchDB atop PostgreSQL using JSONB columns
to see how it would perform. I never ended up taking any of those prototypes
very far, but the thought process helped me appreciate and understand CouchDB's
inner workings better.

One day while working on [Cosmic Verge][cosmicverge], it dawned on me: I had
built a small wrapper around [Sled][sled] that made it feel like a higher-level
database. What if I went further and built a CouchDB-like API atop Sled?

### {{ anchor(text = "The first year: Initial Goals") }}

Initially, I was hesitant to tell anyone I was building a database. Despite
relying on [Sled][sled] for the low-level transactional storage, it still felt
like one of those projects that is usually ill-advised. I finally felt confident
enough in my reasoning of *why* I was building [BonsaiDb][bonsaidb] that I [wrote a detailed
devlog][pliantdb-devlog-1] describing the motivation for tackling this new
project.

Early on, I never had high ambitions for BonsaiDb. After all, this was the first
database I had ever written. The likelihood of me building a high-performance
database seemed low. Instead, I wanted to focus primarily on developer
experience, reliability, and ease of deployment. I tried to make intelligent
choices based on my expectations of what would be efficient, but I intentionally
avoided comparing BonsaiDb against other databases until [January of this
year][benchmarking-blog-post].

To me the selling points of BonsaiDb were (and still are):

- Straightforward to use and deploy.
- Can write generic code that works regardless of how BonsaiDb is deployed.
- "Battery-powered" with common needs for developers, reducing complexity of deployments:
  - ACID-compliant transactional document storage
  - Lightweight Atomic Key-Value store (a-la Redis)
  - PubSub
  - Unified and extensible authentication and permissions model
  - Persistent Job Queue System (a la SQS/Resque/Sidekiq). Work on this has been
    started but is not shipping yet, but it has been a goal from the start of
    the project.
- Built in Rust, for Rust.
- Vision includes a "grows with you" pitch:
  - Bi-directional replication
  - Quorum-based clustering
  - Modular plugin system to support "resuable" collections and services

### {{ anchor(text = "BonsaiDb isn't just a toy") }}

In October a series of events made me look at porting the append-only B-Tree
implementation from CouchDB to Rust as a potential replacement for [Sled][sled]
in our architecture. In the end, the bug that I was experiencing on Mac [was
unrelated][memory-bug] to Sled, but its symptoms manifested in such a way that
resembled issues others have had with Sled in the past.

After getting some basic unit tests working, I did something really stupid: I
benchmarked it against Sled and SQLite. Why was it stupid? It kicked off a
viscious few weeks, nearly reaffirming the common wisdom: why was I trying to
write my own database? I would stare [at microbenchmarks][nebari-benchmarks] and
profiling results, tweak something, and try again. Over and over. For some
reason it just wasn't occurring to me: the fact [Nebari][nebari] was even in the
same realm as these highly optimized databases was exciting enough. Eventually,
I [convinced myself][why-nebari] it was better for the future of BonsaiDb to
adopt Nebari.

In January I was preparing to release our first alpha. I knew if I didn't have
benchmarks available, it would be one of the first questions asked. I wrote a
benchmark suite aimed at comparing BonsaiDb against PostgreSQL in a simple
eCommerce setup. I wasn't prepared for the results: BonsaiDb is significantly
faster than PostgreSQL [in this benchmark suite][commerce-bench-results].

Last month I set my focus [testing large datasets][large-datasets] and was able
to achieve performance again that exceeded my expectations. I no longer am
viewing BonsaiDb as an easy-to-use database that will be "good enough" for many
people. I now view BonsaiDb as having the potential to be a serious database
option for most people.

For being just one year old, I'm extremely happy with where BonsaiDb is today.

## {{ anchor(text = "Year Two: My Goals") }}

One thing that surprised me was how much interest [Nebari][nebari] garnered.
Despite not heavily advertising it, it's accumulated a fair number of stars on
GitHub. It's popular enough that I've been rethinking how I was planning on
implementing certain features. Outside of bug fixes and minor features, I
haven't spent much time working on Nebari as it has been meeting BonsaiDb's
needs. I would like to spend more time in this coming year making Nebari a
better crate for developers to consume directly. Ultimately, the more users
Nebari has, the more confidence everyone can have in BonsaiDb's ACID compliance.

For [BonsaiDb][bonsaidb], there are several goals I have:

- **Features, Features, Features**: I'm excited at exploring a wide variety of
  features aiming to make BonsaiDb more powerful and easier to use:

  - [Ability to use BonsaiDb in non-async
    code](https://github.com/khonsulabs/bonsaidb/pull/220)
  - [Persistent Job Queue](https://github.com/khonsulabs/bonsaidb/issues/78)
  - [Supporting Metrics](https://github.com/khonsulabs/bonsaidb/issues/162)
  - [Document Dependencies/Foreign Keys/Graph Relationships](https://github.com/khonsulabs/bonsaidb/issues/136)
  - [Collection Lifecycles](https://github.com/khonsulabs/bonsaidb/issues/69)
  - [Replication](https://github.com/khonsulabs/bonsaidb/issues/90)

  There are countless other features I hope to explore as well, but those are
  some of the higher priority items as I view them.

- **Stability**: I would like to be able to declare a stable version within the
  next year. I'm currently treating BonsaiDb as if it's stable: trying to
  preserve backwards compatibility for the storage format, prioritizing bug
  fixes, and keeping a thorough changelog.

  By this time next year, I would like to be at a point where the core of
  BonsaiDb is changing infrequently, and focus is on building higher level
  abstractions.

- **Tooling**: A good, graphical database administration tool not only makes
  exploring a database more approachable, it also can aide in quickly diagnosing
  and fixing data issues. Because BonsaiDb is architected to not know about the
  contents of the documents being stored, creating a good, generic
  administration tool will be a fun challenge.

- **Community**: Fostering an active developer community around BonsaiDb is
  important for many reasons. As I mentioned before, the more users BonsaiDb
  has, the more confident we can all feel about its reliability. Beyond that,
  however, is something more fundamental for me: I thrive on success stories and
  helping solve interesting data problems.
  
## {{ anchor(text = "Help grow BonsaiDb") }}

Building BonsaiDb has reawakened many of the passions I felt in my first
full-time computer-related job: working on REALbasic (now
[Xojo](https://www.xojo.com/)). As I started getting schema design questions and
feature requests from potential users, I reflected on the joy those interactions
provided me. It is clear to me that one of the reasons I look back so fondly at
my first programming job is the wide array of problems I helped users solve.

For the longest time, I held off on allowing any means of sponsoring development
of BonsaiDb. Until recently, I was still hoping to get back to writing an MMO
once BonsaiDb was mature enough. My dreams have changed in the past months: I
want to focus on bringing safe, high-level data-oriented application development
to Rust. Initially, that means getting BonsaiDb to a maturity level where it
meets my original vision.

Many readers may not be aware that the early days of BonsaiDb development were
overlapped with my development of [Gooey][gooey]. To this day, it is the only
framework that meets my personal desires of application development. However,
it's lacking many critical features/widgets. I paused development because I want
to focus on BonsaiDb. Yet, I also want to begin writing GUI administration tools
for BonsaiDb soon.

I would love to continue dedicating my focus to these areas of the Rust ecosystem:

- BonsaiDb: Easy-to-use database that grows with you.
- Building cross-platform applications in Rust (with or without Gooey)

If you would like to ensure my ability to continue working on these projects
full-time, I am now [accepting sponsorship through GitHub
Sponsors][ecton-sponsor]. I have been working on open-source Rust projects since
November 2019 all funded by the sale of my previous startup. I would love the
opportunity to continue working on open-source full time without needing to
focus on building another startup. As I am not in dire need of finances, please
only sponsor me if you have truly disposable income.

Another important way to help grow BonsaiDb is to contribute. I've been working
on adding ["Good First Issue"][first-issues] tasks. If you're looking to
contribute to an open-source Rust project, I would be honored to have you as
part of our team.

## {{ anchor(text = "Trying out BonsaiDb") }}

Our [homepage](https://bonsaidb.io/) has basic setup instructions and a list of
examples. We have started writing a [user's
guide](https://dev.bonsaidb.io/release/guide/), and we have tried to write [good
documentation](https://docs.rs/bonsaidb/).

I would love to hear from you if you have questions or feedback. We have
[community Discourse forums](https://community.khonsulabs.com/) and a [Discord
server](https://discord.khonsulabs.com), but also welcome anyone to [open an
issue](https://github.com/khonsulabs/bonsaidb/issues/new) with any questions or
feedback.

Lastly, if you build something with one of our libraries, we would love to hear
about it. Nothing makes us happier than hearing people are building things with
our crates!

Thank you to all of the wonderful people in the Khonsu Labs community and the
Rust ecosystem.

[bonsaidb]: https://github.com/khonsulabs/bonsaidb
[initial-commit]: https://github.com/khonsulabs/bonsaidb/commit/43bd3a25b61fc7841c9554422d7bb46ad4362e59
[ecton]: https://github.com/ecton
[ecton-sponsor]: https://github.com/sponsors/ecton
[daxpedda]: https://github.com/daxpedda
[fabruic]: https://github.com/khonsulabs/fabruic
[modprog]: https://github.com/modprog
[vbmade2000]: https://github.com/vbmade2000
[name-change]: https://community.khonsulabs.com/t/short-update-pliantdb-is-now-known-as-bonsaidb/74
[couchdb]: https://couchdb.apache.org/
[pliantdb-devlog-1]: https://community.khonsulabs.com/t/did-i-say-i-was-refocusing/58#pliantdb-the-core-of-cosmic-verge-2
[benchmarking-blog-post]: https://bonsaidb.io/blog/commerce-benchmark/
[sled]: https://sled.rs
[memory-bug]: https://community.khonsulabs.com/t/introducing-nebari-a-key-value-data-store-written-using-an-append-only-file-format/81#what-caused-the-uncontrolled-memory-usage-5
[nebari]: https://github.com/khonsulabs/nebari
[nebari-benchmarks]: https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/nebari-scaleway-gp1-xs/index.html
[why-nebari]: https://community.khonsulabs.com/t/introducing-nebari-a-key-value-data-store-written-using-an-append-only-file-format/81#why-still-develop-nebari-6
[commerce-bench-results]: https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/commerce/index.html
[large-datasets]: https://bonsaidb.io/blog/february-2022-update/#Testing%20BonsaiDb%20with%20a%20large%20dataset
[gooey]: https://github.com/khonsulabs/gooey/
[first-issues]: https://github.com/khonsulabs/bonsaidb/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22
[cosmicverge]: https://github.com/khonsulabs/cosmicverge
