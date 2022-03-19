+++
title = "BonsaiDb January update: Alpha Next Week"

[extra]
author = "Jonathan Johnson"
author_url = "https://github.com/ecton"
+++

[BonsaiDb][bonsaidb] is a pure Rust database that grows with you. It already is [feature-rich](/about), but we are still working towards our initial alpha. This month's update will highlight the changes since the last update and cover what's remaining before our alpha release.

## Highlighted updates

I have made [90 commits (+8,406/-3,097)][changes] since [last month's update][last-update]. Many improvements have been made based on feedback from early adopters. Thank you to everyone who has asked a question, reported an issue, or provided any feedback!

### {{ anchor(text = "Key-Value Store Optimization") }}

I firmly believe in under-promising and over-delivering. So when I was writing the [about page](/about) originally, I had to clarify in the [Key-Value section](/about/#key-value) that the performance was limited by [the current design][key-value-issue]. As such, I decided I would prefer to go ahead and implement the design proposed in that issue.

The Key-Value store is meant to perform atomic operations with reasonable durability. This is the primary difference between the goals of the Key-Value store and [Collection][collection] storage. The Key-Value store is meant to be a good alternative to running [Redis][redis], and Redis uses a delayed persistence design.

I've copied the general design of how persistence works, except that with BonsaiDb the database isn't kept in memory constantly. This means that BonsaiDb's Key-Value store can contain more data than the RAM on your machine, something that is possible but not recommended with Redis.

By default, BonsaiDb will persist each operation after it's performed. This can cause a lot of extra disk IO if thousands of operations are being performed per second. Additionally, it can cause file bloat due to the [append-only file format][nebari] utilized. Instead, it's recommended to [configure persistence][kv-configuration] such that writes are delayed based on what you feel is a good balance for your use case. For example:

```rust
Storage::open(
    StorageConfiguration::new(&database_directory)
        .key_value_persistence(KeyValuePersistence::lazy([
            PersistenceThreshold::after_changes(1)
    .and_duration(Duration::from_secs(5)),
            PersistenceThreshold::after_changes(10),
  ])
)
```

This configuration has two rules: persist after 5 seconds if there is at least 1 change, and perist if there are at least 200 changes. This means if BonsaiDb unexpectedly is killed, at most the most recent 5 seconds of changes would be lost. However, if a batch of changes is written, they will be persisted immediately.

So, how does the key-value store perform compared to Redis?

[{{ blockimage(alt="get-key 1kb", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/get-bytes/1KiB/report/violin.svg", bg = true )}}](https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/suite/get-bytes/1KiB/report/index.html)

Well, as you might expect, if you don't have network access, things go very fast -- measured in nanoseconds on my personal computer. However, the networking performance leaves something to be desired. After doing a lot of profiling, I could see that the TLS for the QUIC connection accounts for roughly 30% of the time spent. However, that still is a little slower than the WebSocket implementation, which in turn is significantly slower than Redis.

My profiling has led me to believe that [switching to Socketto][soketto] will bring the WebSocket implementation closer by reducing the number of allocations. Only time will tell if that will match Redis's performance, but I'm hopeful it will be close enough to not care. For the QUIC connection, our [other major contributor][daxpedda] has plans to dig in and see what can be done to reduce some of the allocations we saw in the profiling we did.

### {{ anchor(text = "Collection Benchmarks") }}

My last [blog post][commerce-blog] goes into detail about a new benchmark I wrote to attempt to simulate a simple relational database workload. The results were staggering to me.

[{{ blockimage(alt="find prodcut by id graph", src="https://khonsulabs-storage.s3.us-west-000.backblazeb2.com/bonsaidb-scaleway-gp1-xs/commerce/large-writeheavy/8/LookupProduct.png" )}}][commerce-blog]

Nearly every operation across every workload showed all ways of accessing BonsaiDb outperforming PostgreSQL significantly.

### {{ anchor(text = "Rethinking Serialization") }}

One problem that I had with BonsaiDb is that I didn't want to force users into a specific serialization format. The base document type contains a buffer of bytes -- you can store whatever you'd like as a document. I believe our new serialization format, [Pot][pot], is a great choice, but there are very good reasons to make it easy to support any serialization format.

This month I spent time creating [Transmog][transmog], my take on an approach to universal serialization traits. This project aims at creating a common interface that most serialization formats could offer an implementation of. By leveraging transmog, BonsaiDb's [SerializedCollection][serializedcollection] trait can be used to automatically serialize and deserialize your collection types for you regardless of the format you're storing them as. Additionally, View value serialization is now powered by Transmog, as our [hdrhistogram view example][hdrhistogram-example] demonstrates:

```rust
/// A view for [`Samples`] which produces a histogram.
#[derive(Debug, Clone)]
pub struct AsHistogram;

impl View for AsHistogram {
    type Collection = Samples;
    type Key = u64;
    type Value = SyncHistogram<u64>;

    fn name(&self) -> Name {
        Name::new("as-histogram")
    }
}

impl SerializedView for AsHistogram {
    type Format = Self;

    fn format() -> Self::Format {
        Self
    }
}

impl Format<'static, SyncHistogram<u64>> for AsHistogram {
    type Error = HistogramError;

    fn serialize_into<W: std::io::Write>(
        &self,
        value: &SyncHistogram<u64>,
        mut writer: W,
    ) -> Result<(), Self::Error> {
        V2Serializer::new()
            .serialize(value, &mut writer)
            .map_err(HistogramError::Serialization)?;
        Ok(())
    }
}

impl OwnedDeserializer<SyncHistogram<u64>> for AsHistogram {
    fn deserialize_from<R: std::io::Read>(
        &self,
        mut reader: R,
    ) -> Result<SyncHistogram<u64>, Self::Error> {
        hdrhistogram::serialization::Deserializer::new()
            .deserialize(&mut reader)
            .map(SyncHistogram::from)
            .map_err(HistogramError::Deserialization)
    }
}
```

This example shows how using [Transmog][transmog] we're able to use the custom serialization functions built into the [hdrhistogram][hdrhistogram] crate. Querying the view returns a `SyncHistogram<u64>` directly:

```rust
let total_histogram = db.view::<AsHistogram>().reduce().await?;
println!(
 "99th Percentile overall: {} ({} samples)",
 total_histogram.value_at_quantile(0.99),
 total_histogram.len()
);
```

I think this example shows the inredible power of our [map/reduce views][views] and the Rust type system.

## {{ anchor(text = "Alpha coming next week") }}

I had a chat this morning with our other major contributor, ([@daxpedda][daxpedda]), to try to understand the implications of the breaking changes coming in the [OPAQUE-KE][opaque-ke] crate. The result of the conversation was [this issue to implement traditional password hashing][password-hashing]. The first alpha will not contain OPAQUE as we're going to wait until an update supporting  draft-irtf-cfrg-opaque-07 is released. This is one of the things he's been working hard on for months, and he's near the point where we can integrate the changes into BonsaiDb!

However, as OPAQUE is still a draft specification, there is too much possibility that things will break again in the future before being stabilized. We will help support these upgrades, but it's not going to be the best administration experience, and it will potentially require utilizing multiple crate versions in the long run.

Once I swap out this implementation, I'm going to be creating a 0.1 branch [in the repository][bonsaidb]. We will be using the 0.x version range for our alpha and beta phases and eventually [release 1.0][stable] as the first stable version.

I will ask early adopters to check it out ahead of releasing to crates.io at the end of next week. I'll send out a message on [our Discord][discord] as well as update [this issue][alpha-issue] once the branch is available.

In the meantime, our [homepage](/) has basic getting started information including a full list of examples. I look forward to hearing what people build with BonsaiDb!

> Have questions or comments? Discuss this post on [our forums](https://community.khonsulabs.com/t/bonsaidb-january-update-alpha-next-week/93).

[changes]: https://github.com/khonsulabs/bonsaidb/compare/355c7904dd9b64874d99721941d2b0c0002f26b4...c1bc3ca6ce488fe8c26d265a3b1e9b8fb62d1347
[bonsaidb]: https://github.com/khonsulabs/bonsaidb
[nebari]: https://github.com/khonsulabs/nebari
[last-update]: https://community.khonsulabs.com/t/bonsaidb-december-update-finishing-up-alpha-1/88
[commerce-blog]: https://bonsaidb.io/blog/commerce-benchmark/
[key-value-issue]: https://github.com/khonsulabs/bonsaidb/issues/120
[collection]: https://dev.bonsaidb.io/release/guide/about/concepts/collection.html
[kv-configuration]: https://dev.bonsaidb.io/release/guide/administration/configuration.html#key-value-persistence
[soketto]: https://github.com/khonsulabs/bonsaidb/issues/129
[pot]: https://github.com/khonsulabs/pot
[transmog]: https://github.com/khonsulabs/transmog
[serializedcollection]: https://dev.bonsaidb.io/release/docs/bonsaidb/core/schema/trait.SerializedCollection.html
[hdrhistogram-example]: https://github.com/khonsulabs/bonsaidb/blob/main/examples/view-histogram/examples/view-histogram.rs
[hdrhistogram]:  https://crates.io/crates/hdrhistogram
[views]: https://dev.bonsaidb.io/release/guide/about/concepts/view.html
[opaque-ke]: https://github.com/novifinancial/opaque-ke
[password-hashing]: https://github.com/khonsulabs/bonsaidb/issues/158
[daxpedda]: https://github.com/daxpedda
[discord]: https://discord.khonsulabs.com/
[alpha-issue]: https://github.com/khonsulabs/bonsaidb/issues/87
[redis]: https://redis.io/
[stable]: https://github.com/khonsulabs/bonsaidb/issues/137
