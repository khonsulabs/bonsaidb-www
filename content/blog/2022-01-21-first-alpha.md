+++
title = "Announcing BonsaiDb's First Alpha Release"
+++

It's been almost six months since [we announced PliantDb 0.1.0-dev.4](https://community.khonsulabs.com/t/pliantdb-v0-1-0-dev-4-released-custom-apis-unique-views-webassembly/69). Beyond the name change, a lot has changed. We've been focusing our efforts on stabilizing BonsaiDb's core architecture so that we can start building applications with BonsaiDb.

[BonsaiDb][bonsaidb] is a developer-friendly document database that grows with you. It can work offline in a local-only mode or operate in a networked client/server mode. In addition, it will eventually support replication and clustering. To learn more about BonsaiDb, [see this feature overview](https://bonsaidb.io/about).

## Upgrading existing databases

There is no upgrade path from 0.1.0-dev.4. We have warned against using this project outside of experimentation, so we hope this is not an issue for anyone. However, if this is devastating news to anyone, please reach out to us, and we can try to help.

## What does alpha mean?

We only released the first alpha once we felt reasonably confident we had a solid foundation. We expect bugs will be found, and we are certain that at least one user will lose data. It's just a fact of life. The alpha phase enables adventurous users to begin using BonsaiDb to help us find these issues before we label BonsaiDb stable.

We encourage users that attempt to use this in an environment where data loss would be detrimental to utilize and test the [backup and restore](https://bonsaidb.io/about/#backup-restore) functionality regularly.

We are still planning to add features during the alpha phase, but those features will focus on the higher-level database functionality. Changes to how data is stored will be minimal or non-existent unless bugs or performance issues are uncovered.

## What's new?

In many ways, BonsaiDb v0.1.0-alpha.1 is an entirely different database than PliantDb v0.1.0-dev.4. And yet, in other ways, it hasn't changed much. The [CHANGELOG](https://github.com/khonsulabs/bonsaidb/blob/main/CHANGELOG.md) covers all of the changes, but we wanted to draw attention to a few notable entries.

### New Storage Layer

Under the hood, this alpha unveils a new storage engine: [Nebari](https://github.com/khonsulabs/nebari). While the storage layer we were using before ([Sled](https://sled.rs)) is incredibly fast, it also is quite complex and didn't quite fit BonsaiDb's needs perfectly.

Nebari maintains a fairly straightforward design while still achieving fairly competitive benchmarks for being such a new library. Additionally, its implementation can be custom-tailored to the workloads we need to support BonsaiDb.

### New Serialization Ecosystem

Part of our design from the beginning was to allow users to store arbitrary chunks of data as a [document](https://dev.bonsaidb.io/guide/about/concepts/document.html). In other words, users shouldn't be forced into a specific serialization or encoding format for their data. Unfortunately, a side effect of this was the API's usability suffered.

This alpha addresses the API shortcomings with functionality designed around [Transmog](https://github.com/khonsulabs/transmog). Transmog is a fairly simple concept: a universal set of traits to describe a basic serialization and deserialization interface. An example of Transmog in cation can be found in our new example: [view-histogram.rs](https://github.com/khonsulabs/bonsaidb/blob/main/examples/view-histogram/examples/view-histogram.rs). It demonstrates storing a [hdrhistogram::Histogram](https://docs.rs/hdrhistogram/latest/hdrhistogram/struct.Histogram.html) as a view's value by implementing the needed Transmog traits.

Using Transmog as a foundation, BonsaiDb provides streamlined APIs for interacting with [serialized collections](https://dev.bonsaidb.io/guide/about/concepts/document.html#serializable-collections).

### Extensible TCP Handling

We have a vision that BonsaiDb will be a complete application development platform, including building web applications. Therefore, we have created a way to theoretically use any HTTP framework and mount the BonsaiDb WebSockets at a specific route.

We have a new example ([axum.rs](https://github.com/khonsulabs/bonsaidb/blob/main/examples/axum/examples/axum.rs)) which demonstrates how to accomplish this using the [Axum](https://crates.io/crates/axum) framework.

### At-Rest Encryption

Not all hosting environments offer encrypted filesystems, and almost all applications store some Personally Identifiable Information (PII). Therefore, it is a good practice (and often a legal requirement) to ensure that all sensitive information is stored encrypted on the disk.

BonsaiDb now optionally supports transparent [at-rest encryption using externally stored keys](https://dev.bonsaidb.io/guide/administration/encryption.html).

## Improving the Ecosystem

New and updated crates in the Rust ecosystem power this release:

### New Crates

- [custodian-password](https://crates.io/crates/custodian-password): High-level implementation of the OPAQUE password-authenticated key exchange protocol
- [derive-where](https://github.com/ModProg/derive-where/): Derive standard traits more easily when generic type bounds are involved
- [Nebari](https://github.com/khonsulabs/nebari): Low-level, transactional key-value storage
- [ordered-varint](https://github.com/khonsulabs/ordered-varint): Variable-length signed and unsigned integer encoding that retains sortable at the byte-level
- [Pot](https://github.com/khonsulabs/pot): A concise, self-describing binary serialization format.
- [RustMe](https://github.com/khonsulabs/rustme): Tool and library for managing READMEs for Rust projects
- [Transmog](https://github.com/khonsulabs/transmog): Universal serialization traits and utilities.

### Updated Crates

These are existing crates we've worked on as part of developing this release of BonsaiDb:

- [async-acme](https://github.com/User65k/async-acme/): Async ACME client for the TLS-ALPN-01 challenge type
- [opaque-ke](https://github.com/novifinancial/opaque-ke): An implementation of the OPAQUE password-authenticated key exchange protocol. Used by custodian-password.
- [password-hashes](https://github.com/RustCrypto/password-hashes): Password hashing functions and KDFs.
- [voprf](https://github.com/novifinancial/voprf/): An implementation of a verifiable oblivious pseudorandom function. Used by custodian-password.
- [Zeroize](https://crates.io/crates/zeroize): Utilities for zeroing buffers before deallocating.

## Getting Started

Our [new homepage](https://bonsaidb.io/) has basic setup instructions and a list of examples. We have started writing a [user's guide](https://dev.bonsaidb.io/guide/), and we have tried to write [good documentation](https://dev.bonsaidb.io/main/bonsaidb/).

We would love to hear from you if you have questions or feedback. We have [community Discourse forums](https://community.khonsulabs.com/) and a [Discord server](https://discord.khonsulabs.com), but also welcome anyone to [open an issue](https://github.com/khonsulabs/bonsaidb/issues/new) with any questions or feedback.

We dream big with [BonsaiDb][bonsaidb], and we believe that it can simplify writing and deploying complex, data-driven applications in Rust. We would love additional contributors who have similar passions and ambitions.

Lastly, if you build something with one of our libraries, we would love to hear about it. Nothing makes us happier than hearing people are using what we've built.

[bonsaidb]: https://github.com/khonsulabs/bonsaidb