+++
title = "About"
description = "About Raymond — data engineer based in Taipei."
template = "info-page.html"
+++

I'm Raymond — Yu-Chuan Hung — a data engineer based in Taipei.

## Work

I build lakehouse platforms on Spark and Iceberg, running on Kubernetes:

- **Batch & streaming ingestion** — 900M+ records a day into 500+ production
  tables, including CDC streaming (Kafka + Spark Structured Streaming) with
  sub-3-minute freshness.
- **Catalog migration** — moved 800+ Iceberg tables to Apache Polaris
  (Iceberg REST), removing commit failures from non-atomic S3 renames.
- **Developer experience** — YAML-driven pipeline generation that cut
  source onboarding from 6–8 hours to 3 minutes.

Full CV: [PDF](https://assets.raymondhung.dev/cv/Yu-Chuan_Hung_CV.pdf)

## Open source

I contribute to [Apache DataFusion](https://github.com/apache/datafusion)
and [Apache Arrow (arrow-rs)](https://github.com/apache/arrow-rs) in Rust —
15+ merged PRs so far. A few highlights:

- **Join execution** — sort-merge join filter evaluation on NULL-padded rows
  ([#26027](https://github.com/apache/datafusion/pull/26027)),
  `HashJoinExec` probe metrics
  ([#25272](https://github.com/apache/datafusion/pull/25272))
- **Performance** — type-specialized `scatter`, 1.5–2.5× faster
  ([#20498](https://github.com/apache/datafusion/pull/20498));
  cached dictionary slots in aggregation
  ([#25652](https://github.com/apache/datafusion/pull/25652))
- **Spark compatibility** — `size`
  ([#19592](https://github.com/apache/datafusion/pull/19592)) and
  `json_tuple` ([#20412](https://github.com/apache/datafusion/pull/20412))
- **Parquet** — Bloom filter properties builder in arrow-rs
  ([#9877](https://github.com/apache/arrow-rs/pull/9877),
  [write-up](@/posts/2026-11-arrow-rs-bloom-filter-builder.md))

All merged PRs:
[DataFusion](https://github.com/apache/datafusion/pulls?q=is%3Apr+is%3Amerged+author%3ACuteChuanChuan) ·
[arrow-rs](https://github.com/apache/arrow-rs/pulls?q=is%3Apr+is%3Amerged+author%3ACuteChuanChuan)

## Contact

Find me on [GitHub](https://github.com/CuteChuanChuan),
[LinkedIn](https://www.linkedin.com/in/yuchuanhung),
or [by email](mailto:yuchuanhung.raymond@gmail.com).

*Write to remember, publish to be remembered.*
