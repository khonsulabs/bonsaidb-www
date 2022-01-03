+++
title = "What is BonsaiDb?"
+++

BonsaiDb's is a developer-friendly database for the [Rust programming
language][rust] that grows with you and your projects.

A developer-friendly database should be able to solve most of your data storage
and transmission needs. The features BonsaiDb offers are catered to solving some
of the most common problems developers face when creating connected, data-driven
applications.

[rust]: https://rust-lang.org

## BonsaiDb's Features

### It grows with you

BonsaiDb's features are designed to work when using it as a local-only database,
as a networked database server, and eventually [as a distributed
cluster][clustering]. This design makes it easy to test and develop apps with
BonsaiDb, as there are is additional software for developers to set up when
working on a BonsaiDb powered app.

[clustering]: https://github.com/khonsulabs/bonsaidb/issues/104

### Schema as Rust Code

BonsaiDb leverages the Rust type system to allow defining schema as code,
enabling the Rust compiler to help you ensure your database code is correct.
This helps keep the likelihood of unexpected runtime errors to a minimum, and it
also helps developers understand the entire flow of their code and data.

### ACID-compliant Transactional Storage

BonsaiDb's [collections][collection] are [ACID][acid]-compliant, giving you the
peace of mind that your data is safe in the event of an unexpected failure of
the server. For most developers, this mode of storage should be the default, yet
many popular NoSQL database choices do not offer ACID-compliant storage.

An example of where ACID-compliance matters: Storing a customer's order in a web
store. If the method of writing the customer's order to the database is not
ACID-compliant, a power failure could occur between saving the customer's order
and the bytes being persisted to disk. This could result in a user being
notified that their order was successfully submitted (and worse, their credit
card charged), yet after the power is restored, your database has no record of
the transaction.

[collection]: https://dev.bonsaidb.io/guide/about/concepts/collection.html
[acid]: https://en.wikipedia.org/wiki/ACID

### At-Rest Encryption

BonsaiDb optionally supports [encrypting data at-rest][at-rest-encryption],
which prevents data from being leaked if a hard drive is stolen or wasn't wiped
before being removed from a data center. This is an important feature to have
when storing Personally Identifiable Information (PII) if your hosting
environment does not offer ways of encrypting the filesystem itself.

[at-rest-encryption]: https://dev.bonsaidb.io/guide/administration/encryption.html

### Backup / Restore

BonsaiDb offers [backup][backup] and [restore][restore] from a backup location,
which is a trait that can be implemented for custom backup solutions. BonsaiDb
provides built-in support for using a filesystem directory as a backup location,
and there is planned support for any [S3-compatible storage service][s3-backup].
Currently backups are always complete backups, but [incremental backups are
planned][incremental-backups].

[backup]: https://dev.bonsaidb.io/main/bonsaidb/local/struct.Storage.html#method.backup
[restore]: https://dev.bonsaidb.io/main/bonsaidb/local/struct.Storage.html#method.restore
[s3-backup]: https://github.com/khonsulabs/bonsaidb/issues/122
[incremental-backups]: https://github.com/khonsulabs/bonsaidb/issues/121

### Extensible Role-based Access Control

BonsaiDb offers a robust [permission system][permissions] which allows for
"actions" to be allowed or denied on "resources". BonsaiDb uses this access
control system internally, but it is written to be able to be extended by
actions and resources defined in your application.

With BonsaiDb, you can choose to utilize the multi-user support as your account
system for your application. By doing this, you can use roles and groups to
manage your application's permissions, not just the database's permissions.

This feature is still under development, and currently permissions are only
enforced at the network level. There are [plans to allow permissions to be
evaluated offline as well][permissions-refactor].

[permissions]: https://dev.bonsaidb.io/guide/administration/permissions.html
[permissions-refactor]: https://github.com/khonsulabs/bonsaidb/issues/68

### Atomic Key-Value Storage

Sometimes ACID-compliance is overkill, such as in situations where a value is
being updated hundreds or thousands of times per second. BonsaiDb offers a
namespaced, atomic [key-value store][leu=value]. It currently offers basic
atomic operations as well as some atomic arithmetic operations.

This feature is still early in development, and currently is fully
transactional. This means it is currently not very "lightweight", but there are
[plans to address this][key-value-refactor].

[key-value]: https://dev.bonsaidb.io/guide/traits/key-value.html
[key-value-refactor]: https://github.com/khonsulabs/bonsaidb/issues/120

### Publish/Subscribe (PubSub)

BonsaiDb offers the ability to subscribe to topics and receive published
messages. [PubSub][pubsub] can be used to power features like private messaging
but can also be used as a way to separate services within an application's
architecture.

[pubsub]: https://dev.bonsaidb.io/guide/about/concepts/pubsub.html

### Secure Networked Access

BonsaiDb offers two wire protocol implementations, one that utilizes
[QUIC][quic] and one that utilizes [WebSockets][websockets]. The QUIC-based
protocol is more efficient, but to enable access in the web browser, WebSockets
are also offered. WebSockets may eventually be replaced or supplemented with a
WebRTC offering.

[quic]: https://en.wikipedia.org/wiki/QUIC
[websockets]: https://en.wikipedia.org/wiki/WebSocket

#### Extensible Wire Protocol

BonsaiDb offers an ability to extend the wire protocol with a request/response
style API. This provides an easy way to provide authenticated, permission-aware
access to server-side functionality.

The user's guide has [a page dedicated to an example of this setup][custom-api].

[custom-api]: https://dev.bonsaidb.io/guide/about/access-models/custom-api-server.html

#### Extensible TCP Layer

Suppose you're designing an app that exposes an HTTP layer and wish to connect
to your database server over WebSockets. BonsaiDb allows upgrading TCP
connections [using a `hyper::Request`][websocket-upgrade] or [manually after
performing the negotation][websocket-handle].

An example of this configuration is [included in the repository][axum-example].
It shows how to use the [Axum][axum] framework alongside BonsaiDb's WebSockets.

[websocket-upgrade]: https://dev.bonsaidb.io/main/bonsaidb/server/struct.CustomServer.html#method.upgrade_websocket
[websocket-handle]: https://dev.bonsaidb.io/main/bonsaidb/server/struct.CustomServer.html#method.handle_websocket
[axum-example]: https://github.com/khonsulabs/bonsaidb/blob/main/examples/axum/examples/axum.rs
[axum]: https://crates.io/crates/axum

#### TLS Certificates via LetsEncrypt

BonsaiDb requires TLS for its connections. You can use any valid TLS
certificate. If you would like BonsaiDb to automatically acquire one using ACME,
it's as simple as enabling a feature flag and [listening for TCP connections on
port 443][server-listen]. An example showing how this is configured is available
[in the repository][acme-example].

[server-listen]: https://dev.bonsaidb.io/main/bonsaidb/server/struct.CustomServer.html#method.listen_for_secure_tcp_on
[acme-example]: https://github.com/khonsulabs/bonsaidb/blob/main/examples/acme/examples/acme.rs

## BonsaiDb's Planned Features

- [Persistent Job Queue][job-queue]: BonsaiDb needs to run some tasks on a
  scheduled basis, and this schedule should be able to be adjusted by end users.
  Additionally, it's not uncommon for developers to need to run jobs on a
  periodic basis. By hosting these jobs in the database, it enables a
  centralized location for monitoring these jobs. The goal of this system is to
  be similar to [Amazon SQS][sqs] or [Sidekiq][sidekiq].
- [Replication][replication]: Stand up a backup server that replicates some or
  all of your databases. If your primary server fails, start using the backup
  server as the primary server. Or, replicate a production database into a local
  environment to test.

  Replication will catch issues when conflicts arise and provide methods to
  resolve those conflicts.
- [Clustering][clustering]: Once your application has grown enough to worry
  about high availability, clustering will allow you to share the load and
  reliability across a minimum of three servers. We plan to support being able
  to control clustering on a per-database level, and envision the ability to
  deploy a globally distributed cluster with fine control over where your
  databases are placed.

And [more][enhancements]. We welcome feedback and suggestions.

[job-queue]: https://github.com/khonsulabs/bonsaidb/issues/78
[sqs]: https://aws.amazon.com/sqs/
[sidekiq]: https://github.com/mperham/sidekiq
[replication]: https://github.com/khonsulabs/bonsaidb/issues/90
[enhancements]: https://github.com/khonsulabs/bonsaidb/labels/enhancement