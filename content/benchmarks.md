+++
title = "How fast is BonsaiDb?"
+++

BonsaiDb's goal for performance is to be comparable to other popular,
general-purpose, ACID-compliant databases such as SQLite, PostgreSQL, and
MongoDB. We do not consider performance to be a core focus in development and
will make decisions that create a better developer experience over raw
performance.

> This page was written prior [mistakes being discovered](./blog/durable-writes)
> that impacted transactional write performance. Our goals remain the same, but
> the conclusions written below are currently not correct. See [this GitHub
> Issue](https://github.com/khonsulabs/bonsaidb/issues/251) for more information
> on the project to improve transactional write performance.

## {{ anchor(text = "What do these benchmarks tell us about BonsaiDb?") }}

Benchmarking a database is challenging, as each database has a unique set of
features. Microbenchmarks aren't good for evaluating performance, because often
the performance of various functionality changes based on what other load the
database is under. For example, read performance will be impacted by the amount
of writes happening and vice-versa.

The [Commerce Benchmark](/benchmarks/#Commerce%20Benchmark) aims to provide a way to simulate
different ratios of read operations and write operations for a simple eCommerce
schema. It might be tempting to use the results of this benchmark to claim that
BonsaiDb will be faster for your application than another database choice.

The unfortunate reality is that no matter how well designed a benchmark suite
is, it will never be able to predict the performance of a database solution for
your application with your usage patterns.

The only conclusion we want readers to draw from these benchmarks is:
**BonsaiDb's performance is comparable to other mainstream databases.**

## {{ anchor(text = "General Notes") }}

All benchmarks on this page are tailored to try to give a fair comparison
between what functionality BonsaiDb offers and the closest equivalent mode in
the database being compared against. This means that there are ways to operate
other databases in configurations that provide different guarantees (such as not
being fully ACID-compliant) and sometimes those modes can yield faster
execution. These benchmarks will always pick the best way to compare other
databases against BonsaiDb's guarantees, and as such, will not give a complete
picture of any of the databases being compared.

It is important to consider the hardware which BonsaiDb will be deployed to.
Many developers have more powerful development machines than their applications
eventually are deployed on. Each database has slightly different performance
behaviors on various types of hard drives and memory constraints.

## {{ anchor(text = "Commerce Benchmark") }}

The [Commerce Benchmark][commerce-bench-source] attempts to simulate a
relational database workload. All benchmarks are measuring fully ACID-compliant
reads and writes. Click on either graph for the full report.

This suite runs a series of operations with various configurations for:

- The dataset size (small, medium large)
- A load profile (read-heavy, balanced, write-heavy)

For more information about this suite, including instructions on how to run the
suite on your own hardware, see the [benchmark's README][commerce-bench-source].

### Current Results

Results from a [Scaleway](https://scaleway.com) GP1-XS instance running Ubuntu
20.04 with 4 CPU cores, 16GB of RAM, and local NVME storage (only occasionally
updated):

[{{ blockimage(alt="Commerce Benchmark Overview from Scaleway", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/commerce/Overview.png" )}}](https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/commerce/index.html)

Results when executed on GitHub Actions:

[{{ blockimage(alt="Commerce Benchmark Overview from GitHub Actions", src="https://dev.bonsaidb.io/main/benchmarks/commerce/Overview.png" )}}](https://dev.bonsaidb.io/main/benchmarks/commerce/)

[commerce-bench-source]: https://github.com/khonsulabs/bonsaidb/tree/main/benchmarks/benches/commerce

## {{ anchor(text = "Microbenchmarks") }}

BonsaiDb doesn't have many microbenchmarks yet. This suite uses
[Criterion][criterion] for its measurements.

This suite currently only has one benchmark for documents/collections, and a few
for the key-value store.

### Current Results

- Results from a [Scaleway](https://scaleway.com) GP1-XS instance running Ubuntu
20.04 with 4 CPU cores, 16GB of RAM, and local NVME storage (only occasionally
updated): [full report][micro-scaleway]
- Results when executed on GitHub Actions: [full report][micro-github]

Clicking on any graph will take you to the individual benchmark's report page.

#### ACID-compliant Document Inserts

These benchmarks measure the ACID-compliant insert speed of documents of varying
sizes. Random data is utilized for the document contents, which is a worst-case
scenario when enabling compression.

Results from a [Scaleway](https://scaleway.com) GP1-XS instance:

[{{ blockimage(alt="save_documents overview from Scaleway", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/save_documents/report/lines.svg", bg = true )}}](https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/save_documents/report/index.html)

Results when executed on GitHub Actions:

[{{ blockimage(alt="save_documents overview from GitHub Actions", src="https://dev.bonsaidb.io/main/benchmarks/suite/save_documents/report/lines.svg", bg = true )}}](https://dev.bonsaidb.io/main/benchmarks/suite/save_documents/report/index.html)

#### Key-Value Get Bytes

This benchmark measures the atomic get speed from the key-value store with keys
of varying sizes.

Results from a [Scaleway](https://scaleway.com) GP1-XS instance:

[{{ blockimage(alt="get-bytes overview from Scaleway", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/get-bytes/report/lines.svg", bg = true )}}](https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/get-bytes/report/index.html)

Results when executed on GitHub Actions:

[{{ blockimage(alt="get-bytes overview from GitHub Actions", src="https://dev.bonsaidb.io/main/benchmarks/suite/get-bytes/report/lines.svg", bg = true )}}](https://dev.bonsaidb.io/main/benchmarks/suite/get-bytes/report/index.html)

#### Key-Value Set Bytes

This benchmark measures the atomic set (insert) speed from the key-value store
with keys of varying sizes. These operations are not confirmed to be written to
disk before the results are reported, making the key-value store not ACID
compliant.

Results from a [Scaleway](https://scaleway.com) GP1-XS instance:

[{{ blockimage(alt="set-bytes overview from Scaleway", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/set-bytes/report/lines.svg", bg = true )}}](https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/set-bytes/report/index.html)

Results when executed on GitHub Actions:

[{{ blockimage(alt="set-bytes overview from GitHub Actions", src="https://dev.bonsaidb.io/main/benchmarks/suite/set-bytes/report/lines.svg", bg = true )}}](https://dev.bonsaidb.io/main/benchmarks/suite/set-bytes/report/index.html)

#### Key-Value Increment Value

This benchmark measures the atomic increment operation speed from the key-value
store. These operations are not confirmed to be written to
disk before the results are reported, making the key-value store not ACID
compliant.

Results from a [Scaleway](https://scaleway.com) GP1-XS instance:

[{{ blockimage(alt="get-bytes overview from Scaleway", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/increment/report/violin.svg", bg = true )}}](https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/increment/report/index.html)

Results when executed on GitHub Actions:

[{{ blockimage(alt="get-bytes overview from GitHub Actions", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/increment/report/violin.svg", bg = true )}}](https://dev.bonsaidb.io/main/benchmarks/suite/get-bytes/report/index.html)

[micro-scaleway]: https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/report/index.html
[micro-github]: https://dev.bonsaidb.io/main/benchmarks/suite/report/index.html
[criterion]: https://github.com/bheisler/criterion.rs
