+++
title = "What is BonsaiDb?"
+++

BonsaiDb's is a developer-friendly database for the [Rust programming
language](https://rust-lang.org) that grows with you and your projects.

A developer-friendly database should be able to solve most of your data storage
and transmission needs. The features BonsaiDb offers are catered to solving some
of the most common problems developers face when creating connected, data-driven
applications.

## BonsaiDb's Features

### It grows with you

BonsaiDb's features are designed to work when using it as a local-only database,
as a networked database server, and eventually [as a distributed
cluster](https://github.com/khonsulabs/bonsaidb/issues/104). This design makes
it easy to test and develop apps with BonsaiDb, as there are is additional
software for developers to set up when working on a BonsaiDb powered app.

### Schema as Rust Code

BonsaiDb leverages the Rust type system to allow defining schema as code,
enabling the Rust compiler to help you ensure your database code is correct.
This helps keep the likelihood of unexpected runtime errors to a minimum, and it
also helps developers understand the entire flow of their code and data.

### ACID-compliant Transactional Storage

BonsaiDb's
[collections](https://dev.bonsaidb.io/guide/about/concepts/collection.html) are
[ACID](https://en.wikipedia.org/wiki/ACID)-compliant, giving you the peace of
mind that your data is safe in the event of an unexpected failure of the server.
For most developers, this mode of storage should be the default, yet many
popular NoSQL database choices do not offer ACID-compliant storage.

An example of where ACID-compliance matters: Storing a customer's order in a web
store. If the method of writing the customer's order to the database is not
ACID-compliant, a power failure could occur between saving the customer's order
and the bytes being persisted to disk. This could result in a user being
notified that their order was successfully submitted (and worse, their credit
card charged), yet after the power is restored, your database has no record of
the transaction.

### Backup / Restore



### Extensible Role-based Access Control

BonsaiDb offers a robust [permission
system](https://dev.bonsaidb.io/guide/administration/permissions.html) which
allows for "actions" to be allowed or denied on "resources". BonsaiDb uses this
access control system internally, but it is written to be able to be extended by
actions and resources defined in your application.

With BonsaiDb, you can choose to utilize the multi-user support as your account
system for your application. By doing this, you can use roles and groups to
manage your application's permissions, not just the database's permissions.

This feature is still under development, and currently permissions are only
enforced at the network level. There are [plans to allow permissions to be
evaluated offline as well](https://github.com/khonsulabs/bonsaidb/issues/68).

### Atomic Key-Value Storage

Sometimes ACID-compliance is overkill, such as in situations where a value is
being updated hundreds or thousands of times per second. BonsaiDb offers a
namespaced, atomic [key-value
store](https://dev.bonsaidb.io/guide/traits/key-value.html). It currently offers
basic atomic operations as well as some atomic arithmetic operations.

This feature is still early in development, and currently is fully
transactional. This means it is currently not very "lightweight", but there are
[plans to address this](https://github.com/khonsulabs/bonsaidb/issues/120).

### Publish/Subscribe (PubSub)

BonsaiDb offers the ability to subscribe to topics and receive published
messages. [PubSub](https://dev.bonsaidb.io/guide/about/concepts/pubsub.html) can
be used to power features like private messaging but can also be used as a way
to separate services within an application's architecture.

### Secure Networked Access

BonsaiDb offers two wire protocol implementations, one that utilizes
[QUIC](https://en.wikipedia.org/wiki/QUIC) and one that utilizes
[WebSockets](https://en.wikipedia.org/wiki/WebSocket). The QUIC-based protocol
is more efficient, but to enable access in the web browser, WebSockets are also
offered. WebSockets may eventually be replaced or supplemented with a WebRTC
offering.

#### Extensible Wire Protocol

BonsaiDb offers an ability to extend the wire protocol with a request/response
style API. This provides an easy way to provide authenticated, permission-aware
access to server-side functionality.

The user's guide has [a page dedicated to an example of this
setup](https://dev.bonsaidb.io/guide/about/access-models/custom-api-server.html).

#### Extensible TCP Layer

Suppose you're designing an app that exposes an HTTP layer and wish to connect
to your database server over WebSockets. BonsaiDb allows upgrading TCP
connections [using a
`hyper::Request`](https://dev.bonsaidb.io/main/bonsaidb/server/struct.CustomServer.html#method.upgrade_websocket)
or [manually after performing the
negotation](https://dev.bonsaidb.io/main/bonsaidb/server/struct.CustomServer.html#method.handle_websocket).

An example of this configuration is [included in the
repository](https://github.com/khonsulabs/bonsaidb/blob/main/examples/axum/examples/axum.rs).
It shows how to use the [Axum](https://crates.io/crates/axum) framework
alongside BonsaiDb's WebSockets.

#### TLS Certificates via LetsEncrypt

BonsaiDb requires TLS for its connections. You can use any valid TLS
certificate. If you would like BonsaiDb to automatically acquire one using ACME,
it's as simple as enabling a feature flag and [listening for TCP connections on
port
443](https://dev.bonsaidb.io/main/bonsaidb/server/struct.CustomServer.html#method.listen_for_secure_tcp_on).
An example showing how this is configured is available [in the
repository](https://github.com/khonsulabs/bonsaidb/blob/main/examples/acme/examples/acme.rs).

## BonsaiDb's Planned Features

- [Persistent Job Queue](https://github.com/khonsulabs/bonsaidb/issues/78):
  BonsaiDb needs to run some tasks on a scheduled basis, and this schedule
  should be able to be adjusted by end users. Additionally, it's not uncommon
  for developers to need to run jobs on a periodic basis. By hosting these jobs
  in the database, it enables a centralized location for monitoring these jobs.
  The goal of this system is to be similar to [Amazon
  SQS](https://aws.amazon.com/sqs/) or
  [Sidekiq](https://github.com/mperham/sidekiq).
- [Replication](https://github.com/khonsulabs/bonsaidb/issues/90): Stand up a
  backup server that replicates some or all of your databases. If your primary
  server fails, start using the backup server as the primary server. Or,
  replicate a production database into a local environment to test.

  Replication will catch issues when conflicts arise and provide methods to
  resolve those conflicts.
- [Clustering](https://github.com/khonsulabs/bonsaidb/issues/104): Once your
  application has grown enough to worry about high availability, clustering will
  allow you to share the load and reliability across a minimum of three servers.
  We plan to support being able to control clustering on a per-database level,
  and envision the ability to deploy a globally distributed cluster with fine
  control over where your databases are placed.

And [more](https://github.com/khonsulabs/bonsaidb/labels/enhancement). We
welcome feedback and suggestions.
