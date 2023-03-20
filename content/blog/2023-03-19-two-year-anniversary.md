+++
title = "Two Years of BonsaiDb: A retrospective and looking to the future"

[extra]
author = "Jonathan Johnson"
author_url = "https://github.com/ecton"
+++

{{ whatis() }}

Today marks two years since the [initial commit][initial-commit] to
[BonsaiDb][bonsaidb]. It has also been nearly a full year since [the last
release][v0.4.1]. I want to take some time to highlight the major events for
BonsaiDb over the past year, as well as discuss what is coming for BonsaiDb in
the next year.

## Why hasn't there been a new release?

Last May, I [discovered mistakes in my benchmarks](/blog/durable-writes) and set
off on a journey to [improve BonsaiDb's transactional performance][status].
BonsaiDb and its underlying architecture are largely maintained by [me][me]. I
focused on trying to solve the speed issue by creating a new storage layer, and
in the meantime, I had been continuing to work on BonsaiDb off and on.

Last October I hit some burnout, and I thought it was time to try to release an
update to BonsaiDb without the new storage layer. While trying to update the
currently-hidden benchmarks page, I was running into a panic that I only could
reproduce on a virtual machine after letting it run for nearly an hour. My
attempts to simplify were often met with the panic vanishing -- a classic
symptom of a "rare" race condition. Simply running the benchmark with `debug =
true` in the `Cargo.toml` prevented the bug from occurring.

When coupled with the burnout I had at the time, I needed a break. At the
beginning of January, I released the first version of
[OkayWAL](/blog/introducing-okaywal/), and in early February, I
finished my rewrite of [Sediment][sediment] (more on that later). Still, I knew
there was a lot of [remaining work][status].

So I resolved to diagnose and fix the issue. It took a few days, but I
identified a bug that I had already fixed in [Nebari][nebari]'s `main` branch. I
released [Nebari v0.5.5 with the fix backported][v0.5.5]. This finally unblocked
my ability to release an update: no known issues!

Since then, I've been working on a couple of projects [dogfooding][dogfooding]
BonsaiDb. One of those projects, [Dossier][dossier], served the page you're
reading! Not only has it been fun using my own database, it has helped me find a
few rough edges and fix them in the process. I'm also feeling more and more
confident in [the huge list of unreleased changes][changelog].

## When will BonsaiDb v0.5 be available?

I'm currently hoping to have a new version of BonsiaDb released in the next few
weeks. Some notable changes include:

- Feature flag unification: Using Rust 1.60's new dependency syntax, BonsaiDb's
  feature flags are significantly simpler to use.
- [Large file storage support][bonsaidb-files]: Efficiently store files inside
   of your database. As mentioned previously, this page you're reading is served
   with this unreleased feature.
- Significant changes to the `Key` trait and composite keys, including the
  ability to derive the trait. These changes make using custom primary keys or
  view keys significantly easier and more efficient.
- Views can now have their [laziness specified][lazy-views].
- `BLAKE3`-powered token authentication.

I will be publishing a blog post going into more details when the new release is
out.

## Looking back on the year

In last year's anniversary post, I [discussed some goals][goals]. The storage
layer rewrite definitely impacted my goals. Despite this, I wanted to review my
previous goals before talking about what I hope to accomplish this year.

The first goal was to implement a lot of features. Of the list, I only completed
two:

- [Native support for both blocking and
  async](https://github.com/khonsulabs/bonsaidb/pull/220) (available in v0.4)
- [Large file storage support][bonsaidb-files] (will be available in v0.5)

One of the biggest features I really wanted to complete when writing last year's
post was replication. To me, it's somewhat irresponsible for a production
application to run without at least the ability to fail-over to a
"warm-standby". Replication was the path I hoped to enable a form of
[high-availability][high-availability], which was the underlying inspiration of
having replication on the goals list.

I had begun looking at this feature, and realized that to make it efficient, I
would [need to change how documents are
stored](https://github.com/khonsulabs/bonsaidb/issues/225). This work has been
completed in the branch that is working towards [improving transactional write
performance][status]. So, while replication hasn't been completed, some
prerequisite work has been completed.

The second goal from last year was stability.
[KhonsuLabs.com](https://khonsulabs.com) is a site that [is powered by
BonsaiDb](https://github.com/khonsulabs/projects). It's a simple site that shows
off the GitHub activity across all of our projects. I realized the other day I
hadn't even checked to see if it was still running. Sure enough, it's been
working great without any maintenance being done. `systemd` reports that the
service last was launched on `Mon 2023-03-06 10:02:50 UTC` -- over a year of
uptime!

The remaining two goals were tooling and community. The tooling for BonsaiDb
hasn't changed much, but there are features and changes to APIs in v0.5 that are
aimed at making more kinds of tooling possible. Our community has been
growing steadily. I must admit I shied away from growing the community. I
hadn't felt comfortable promoting BonsaiDb due to the looming storage rewrite.

Despite my lack of effort, our [Discord server][discord] has continued to grow,
and I find myself hanging out in the `#coworking` channel with people from the
community several times a week. Additionally, BonsaiDb and its related projects
have gained a few more contributors this year. I'm always so grateful when
others take their time to help improve any of our projects.

## Year Three: My Goals

The major goals I have for the next year are:

- **Ship the [storage rewrite][status]**. At this point, I feel like I have all the
  puzzle pieces lined up. I just need to finish putting the puzzle together.

  [Sediment][sediment] is a new storage format whose goal is to provide
  [ACID-compliant][acid] storage of "blobs" of data. Sediment's purpose is to replace
  [Nebari][nebari]'s current append-only file format. Nebari is the library that
  BonsaiDb uses to store data on-disk.

  Sediment is still in development, but it is mostly complete (nearly 95% unit
  test code coverage!), and most importantly, its performance is meeting my
  goals. I am hopeful that after this is all done that BonsaiDb will be
  competitive with other ACID-compliant databases.

- **Release regular BonsaiDb updates and continue building the community**. This
  past year, I initially lost track of something important: BonsaiDb is already
  pretty nice to use. While I know I am biased, I've really enjoyed some of the
  recent [dogfooding][dogfooding] I've been doing. It may sound alarming that
  transactional insert speed is an order of magnitude slower, but we're talking
  about measurements that are fractions of a second long. For many applications,
  the difference in performance will not be impactful.

  One of the benefits of releasing regular updates is that they provide
  opportunities for new people to hear of BonsaiDb for the first time. At this
  early stage of development, I am hoping to find other Rust developers who
  share a similar vision for easily building and deploying data-driven
  applications.

  Not only would it be great to potentially find additional contributors, I see
  increasing the [bus factor][bus-factor] beyond one as a critical step in
  ensuring BonsaiDb's long-term viability.

- **[Eat my own dogfood][dogfooding] more**. In the past few months, I've been
  creating projects using BonsaiDb both to dogfood and have to fun. This year I
  would like to put more projects into production that are built using BonsaiDb.
  I even have some loose plans to start building a pared-down MMO-like game with
  a friend. Regardless of what projects I end up working on, I want to find
  reasons to use BonsaiDb to test it more and find ways to make it better.

- **Continue to work towards high-availability**. I would love to finish
  shipping replication in the next year, but at a minimum, I want to have made
  some progress towards having a warm-standby style high-availibity offering.
  Minimizing downtime when hardware inevitably fails is an important part of
  taking BonsaiDb from a hobby database to an offering worth considering
  seriously.

- **Work towards better tooling**. Being able to browse and edit your database
  without writing custom tooling is important. Because BonsaiDb doesn't have a
  query language, there isn't a simple way to interact with BonsaiDb databases
  outside of writing Rust code. This needs to change to make BonsaiDb easier to
  use, understand, and maintain.

## {{ anchor(text = "Trying out BonsaiDb") }}

Our [homepage](https://bonsaidb.io/) has basic setup instructions and a list of
examples. We have started writing a [user's
guide](https://dev.bonsaidb.io/release/guide/), and we have tried to maintain [good
documentation](https://docs.rs/bonsaidb/).

I would love to hear from you if you have questions or feedback. We have
[community Discourse forums](https://community.khonsulabs.com/) and a [Discord
server](https://discord.khonsulabs.com), but also welcome anyone to [open an
issue](https://github.com/khonsulabs/bonsaidb/issues/new) with any questions or
feedback.

Lastly, if you build something with one of our libraries, we would love to hear
about it. Nothing makes us happier than hearing people are building things with
our crates!

## Want to get involved or follow along more closely?

If you're interested in contributing to an open-source Rust project, I have been
keeping a list of ["Good First Issue"][first-issues] tasks. Working on BonsaiDb
can feel surprisingly high-level. Regardless of your experience level, if you
are interested in contributing, please reach out!

If you want to hear more frequent updates, I post shorter updates occasionally
on [our community Discord][discord]. I also stream in the `#coworking` channel
several times a week. I've also been trying to be [more active on
Mastodon][mastodon].

Regardless of whether you're looking to contribute or are just interested in
something we've written, I am always happy to answer any questions and greatly
appreciate hearing any constructive feedback.

[sediment]: https://github.com/khonsulabs/sediment
[discord]: https://discord.khonsulabs.com/
[mastodon]: https://fosstodon.org/@ecton
[status]: https://github.com/khonsulabs/bonsaidb/issues/251
[initial-commit]: https://github.com/khonsulabs/bonsaidb/commit/43bd3a25b61fc7841c9554422d7bb46ad4362e59
[bonsaidb]: https://github.com/khonsulabs/bonsaidb
[v0.4.1]: https://github.com/khonsulabs/bonsaidb/releases/tag/v0.4.1
[me]: https://github.com/ecton
[v0.5.5]: https://github.com/khonsulabs/nebari/releases/tag/v0.5.5
[nebari]: https://github.com/khonsulabs/nebari
[changelog]: https://github.com/khonsulabs/bonsaidb/blob/62adae0b78283a4906998ef5be2de61a7b191c99/CHANGELOG.md
[dossier]: https://github.com/khonsulabs/dossier
[dogfooding]: https://en.wikipedia.org/wiki/Eating_your_own_dog_food
[bonsaidb-files]: https://github.com/khonsulabs/bonsaidb/issues/222
[goals]: /blog/one-year-anniversary/#Year%20Two%3A%20My%20Goals
[bus-factor]: https://en.wikipedia.org/wiki/Bus_factor
[first-issues]: https://github.com/khonsulabs/bonsaidb/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22
[high-availability]: https://en.wikipedia.org/wiki/High_availability
[lazy-views]: https://github.com/khonsulabs/bonsaidb/issues/208
[acid]: https://en.wikipedia.org/wiki/ACID
