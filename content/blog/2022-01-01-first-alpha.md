+++
title = "Announcing BonsaiDb's First Alpha Release"
+++

It's been just shy of 6 months since [we announced PliantDb 0.1.0-dev.4](https://community.khonsulabs.com/t/pliantdb-v0-1-0-dev-4-released-custom-apis-unique-views-webassembly/69). Beyond the name change, a lot has changed. We focused our efforts on stabilizing BonsaiDb's core architecture in an effort to start building applications with BonsaiDb.

BonsaiDb is a developer-friendly, document database that grows with you. It is designed to work offline in a local-only mode as well as a client/server mode. It will eventually support replication and clustering. To learn more about BonsaiDb, [see this feature overview](https://bonsaidb.io/about).

## Upgrading existing databases

There is no upgrade path from 0.1.0-dev.4. We have warned against using this project outside of experimentation, so we hope this is not an issue for anyone. If this is devastating news to anyone, please reach out to us, and we can try to help.

## What does alpha mean?

For us, we only declared the version alpha 1 once we felt fairly confident we have a solid foundation to build upon. We are confident there will be bugs found, and we are certain at least one user will lose data. It's just a fact of life. The alpha phase is meant to enable adventurous users to begin using BonsaiDb to help us find these issues before we label it stable.

We encourage users that attempt to use this in an environment where data loss would be detrimental to utilize and test the the [backup and restore](https://bonsaidb.io/about/#backup-restore) functionality.

We are still planning on adding features during the alpha phase, but those features will be focused on the higher level database functionality itself. Changes to how data is stored will be minimal or non-existant, unless bugs or performance issues are uncovered.

## What's new?

In many ways, BonsaiDb v0.1.0-alpha.1 is a completely different database than PliantDb v0.1.0-dev.4. And yet in other ways, it hasn't changed much. The [CHANGELOG](https://github.com/khonsulabs/bonsaidb/blob/main/CHANGELOG.md) covers all of the changes, but we wanted to draw attention to a few notable entries.

### New Storage Layer

Under the hood, this alpha unveils a new storage engine: [Nebari](https://github.com/khonsulabs/nebari). While the storage layer we were using before ([Sled](https://sled.rs)) is incredibly fast, it also is quite complex. 

Nebari maintains a fairly straightforward design while still achieving fairly competitive benchmarks for being such a new library. Additionally, its design can be custom-tailored to the workloads that we needed to support with BonsaiDb.

### New Serialization Ecosystem

Part of our design from the beginning was to allow users to store arbitrary chunks of data as a "document". In other words, users shouldn't be forced into a specific serialization or encoding format for their data.

This alpha introduces a new ecosystem we've designed: [Transmog](https://github.com/khonsulabs/transmog). It's a fairly simple concept: a universal set of traits to describe a basic serialization and deserialization interface. One of our new examples ([view-histogram.rs](https://github.com/khonsulabs/bonsaidb/blob/main/examples/view-histogram/examples/view-histogram.rs)) demonstrates storing a [hdrhistogram::Histogram](https://docs.rs/hdrhistogram/latest/hdrhistogram/struct.Histogram.html) as a view's value by implementing the necessary traits to pass through serialization to that crate.

We've also introduced a new serialization format, [Pot](https://github.com/khonsulabs/pot). Pot is a concise, self-describing serialization format. We noticed that our preferred format, CBOR, had one inefficiency: it repeats identifiers each time they occur. For structures that contain arrays of other structures, this can be a fair amount of wasted space. Pot introduces a dynamically allocated symbol table to avoid repeating the same identifier twice.

When using the default serialization, Pot is being used to serialize your data.

### Extensible TCP Handling

We have a vision that BonsaiDb will be a full application development platform, which includes building web applications. We have built a way to theoretically use any HTTP framework and mount the BonsaiDb websockets at a specific route.

We have a new example ([axum.rs](https://github.com/khonsulabs/bonsaidb/blob/main/examples/axum/examples/axum.rs)) which demonstrates how to accomplish this using the [Axum](https://crates.io/crates/axum) framework.

### At-Rest Encryption

Not all hosting environments offer encrypted filesystems, and almost all applications store some Personally Identifiable Information (PII). It is a good practice (and often a legal requirement) to ensure that all sensitive information is stored encrypted on the disk.

BonsaiDb now optionally supports transparent [at-rest encryption using externally stored keys](https://dev.bonsaidb.io/guide/administration/encryption.html).

## Improving the Ecosystem

This release is powered by new and updated crates in the Rust ecosystem:

### New Crates

- [custodian-password](https://crates.io/crates/custodian-password): High-level implementation of the OPAQUE password authenticated key exchange protocol
- [derive-where](https://github.com/ModProg/derive-where/): Derive standard traits more easily when generic type bounds are involved
- [Nebari](https://github.com/khonsulabs/nebari): Low-level, transactional key-value storage
- [ordered-varint](https://github.com/khonsulabs/ordered-varint): Variable-length signed and unsigned integer encoding that retains sortable at the byte-level
- [Pot](https://github.com/khonsulabs/pot): A concise, self-describing binary serialization format.
- [RustMe](https://github.com/khonsulabs/rustme): Tool and library for managing READMEs for Rust projects
- [Transmog](https://github.com/khonsulabs/transmog): Universal serialization traits and utilities.

### Updated Crates

These are existing crates we've worked on as part of developing this release of BonsaiDb:

- [async-acme](https://github.com/User65k/async-acme/): Async ACME client for the TLS-ALPN-01 challenge type
- [opaque-ke](https://github.com/novifinancial/opaque-ke): An implementation of the OPAQUE password authenticated key exchange protocol. Used by custodian-password.
- [password-hashes](https://github.com/RustCrypto/password-hashes): Password hashing functions and KDFs.
- [voprf](https://github.com/novifinancial/voprf/): An implementation of a verifiable oblivious pseudorandom function. Used by custodian-password.
- [Zeroize](https://crates.io/crates/zeroize): Utilities for zeroing buffers before deallocating.

## Getting Started

Our [new homepage](https://bonsaidb.io/) has basic setup instructions and a list of examples. We have started writing a [user's guide](https://dev.bonsaidb.io/guide/), and we have tried to write [good documentation](https://dev.bonsaidb.io/main/bonsaidb/).

If you have questions or feedback, we would love to hear from you. We have [community Discourse forums](https://community.khonsulabs.com/) and a [Discord server](https://discord.khonsulabs.com), but also welcome anyone to [open an issue](https://github.com/khonsulabs/bonsaidb/issues/new) with any questions or feedback.

We dream big with BonsaiDb, and we believe that it can simplify writing and deploying complex, data-driven applications in Rust. We would love additional contributors who have similar passions and ambitions.

Lastly, if you build something with one of our libraries, we would love to hear about it. Nothing makes us happier than hearing people are using what we've built.
