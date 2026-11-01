+++
title = "Making Parquet Bloom Filter Configuration Explicit in arrow-rs"
date = 2026-11-01
description = "The basic concept of Bloom Filter and how to deal with breaking changes in an open-source project, through apache/arrow-rs#9877."

[taxonomies]
tags = ["arrow-rs", "parquet", "bloom-filter", "rust", "open-source"]
categories = ["open-source"]

[extra]
toc = true
katex = true
+++

## What is this PR for?

This PR adds a new builder for configuring the parameters of Bloom Filter. This article introduces the basic concept of Bloom Filter and how to deal with breaking changes in an open-source project.

## Background — What is Bloom Filter

The most intuitive way to determine whether a specific value is inside a bunch of data is to create a HashMap on the data, and the time complexity of checking the target’s existence is $O(1)$. To be specific, this approach needs to store all the data that appeared — the storage space becomes terrible when the variety (i.e., cardinality) of the data is extremely high. Consider that when we have 10 million rows of UUIDs, we might need 360 MB of space to store all of them. Are there any alternatives that can help users look up the existence of a specific value in a fast way without the heavy demand of storage space? The answer is ‘YES’! *Bloom Filter* (BF) is a solution to quickly determine whether the target is inside lots of data or not by approaching this situation with a different mindset. If BF is used to replace the HashMap (10 million rows of UUIDs), only 7.4 MiB of space is needed for storing the bit set (with FPP = 0.05). However, BF trades off accuracy for storage space. The behavior of adding and searching inside BF is depicted below.

About inserting elements into BF: when we start using a BF, a fixed-size bit array is created and contains only `0`s (`m`: number of bits). When users need to insert a value, `k` hash functions are applied to it, which creates `k` hashed values. Then, the remainders of the hashed values divided by the size of the bit array are calculated, which are regarded as the indexes of the bit array. The bits at these indexes are set to `1` if the original value is `0`.

How about checking whether a value is inside BF or not? The target value is applied with the `k` hash functions, which generates `k` hashed values and the remainder values as well. After that, the bits at these remainder indexes of the bit array are retrieved. If there is any `0` inside these retrieved bit values, we can conclude with 100% confidence that the target value does not exist in the BF (no chance of False Negative). On the contrary, if all retrieved bit values are `1`, we can only assume that the target value might have been inserted into BF before rather than ensure the exact existence of the target value. The reason is that these `1`s might come from several values instead of the target value. In other words, using BF reduces the memory space but the cost is the probability of False Positive.

## `FPP` and `NDV`

When utilizing BF, there are two core configurations of Bloom Filter: `FPP` (False Positive Probability) and `NDV` (Number of Distinct Values). `FPP` is the probability that the BF will incorrectly report the existence of an element which is not actually present. For instance, 0.05 means a 5% chance of a false positive. This value is set up by users to meet their needs. If users want to lower the `FPP`, the approach is to have more bits (more `m`) to reduce the possibility of hash collision, i.e., the main cause of false positive. `NDV` is the count of unique values which will be inserted into the BF. In reality, `NDV` comes from estimation mostly, suggesting that it is possible that the value might be overestimated or underestimated, compared with the actual value. If the actual value of `NDV` exceeds the estimated figure, then the `FPP` will rise (i.e., too many false positives). The intuitive solution is to increase the `m`, but if the NDV is set too high, the BF will consume excessive yet unnecessary space without meaningful increase on performance. There are some revisions on BF to solve the wasted space. One of them was applied by Adrian Garcia Badaracco ([@adriangb](https://github.com/adriangb)) to arrow-rs.

## Before this PR

### Previous Progress

Adrian implemented a way to fold the bit space in his great PR to shrink the filter when flushing. At the beginning, a bigger filter is set up (sized for worst-case `NDV`). The size of this filter will be folded down when flushing to meet the required FPP. This improvement means users no longer need to blindly guess `NDV`, and finally reaches an optimized size of filters and the original value of `NDV` becomes the max `NDV`.

### Existing challenges

In the previous implementation, there are three places that can be further enhanced. First of all, `FPP` has its range, but the original struct literal bypasses the existing validation (`BloomFilterProperties { fpp: 1.5, ndv: ... }`). Second, `WriterPropertiesBuilder` did not have a setter function that receives the whole `BloomFilterProperties`. Third, the original visibility of the field `NDV` is public (`pub ndv: u64`), indicating users can define the value of `NDV` directly. The struct `ColumnProperties` has a field: `bloom_filter_ndv_is_set`. When this field is false, `NDV` will be overridden with `max_row_group_row_count`, which is implicitly configured.

```rust
fn resolve_bloom_filter_ndv(&mut self, default_ndv: u64) {
    if !self.bloom_filter_ndv_is_set {
        if let Some(ref mut bf) = self.bloom_filter_properties {
            bf.ndv = default_ndv;
        }
    }
}
```

## Solution - Builder Pattern and its implementation

Instead of writing the PR immediately, I read Adrian’s implementation and spent some time on Bloom Filter and then I listed my plan on this PR and some open questions, for instance, the `::new()` and `::builder()` methods, and `panic` vs. `try_build`. After obtaining Andrew Lamb’s ([@alamb](https://github.com/alamb)) valuable feedback, I started working on this PR, including creating a builder, adding a validation method, and adding a function to set up Bloom Filter properties to `WriterPropertiesBuilder`. Some code snippets are provided below. To avoid breaking change, the first PR (#9877) used a deprecated alias to keep the old name and added tests to ensure the old path is still available. After the first PR was merged, the second PR (#9969) was submitted to make the public `FPP` and `NDV` fields private, which disables the struct literal, causing a breaking change. The split follows the reviewer’s suggestion — no breaking API change immediately, which is good for reviewers.

```rust
fn validate_bloom_filter_fpp(fpp: f64) -> std::result::Result<(), String> {
    if !(fpp > 0.0 && fpp < 1.0) {
        return Err(format!(
            "fpp must be between 0.0 and 1.0 exclusive, got {fpp}"
        ));
    }
    Ok(())
}
```

```rust
// Before
let bf = BloomFilterProperties { fpp: 0.01, ndv: 10_000 };

// After
let bf = BloomFilterProperties::builder()   // or BloomFilterPropertiesBuilder::new()
    .with_fpp(0.01)
    .with_max_ndv(10_000)
    .build();                               // panics on bad fpp; try_build() -> Result

let props = WriterProperties::builder()
    .set_column_bloom_filter_properties(ColumnPath::from("col"), bf)
    .build();
```

## With this PR

In general, I have learned three things when doing this PR. First, organize questions related to the issue and politely ask for reviewers’ opinions and thoughts first, rather than directly writing code and submitting a PR, if there are some concerns about the issue. This action can save several round-trips between contributor and reviewers. Second, instead of including all things (i.e., adding properties builder and making public fields private) in one PR, splitting them into two PRs (one includes adding builder and deprecated warning and the other actually makes public fields private) not only makes the reviewing process easier, but also lets reviewers control the release progress of the breaking change. Third, this PR opened the gate to Bloom Filter, SBBF (Split Block Bloom Filter), and bit-set folding for me when I read [PR #9628](https://github.com/apache/arrow-rs/pull/9628).

## Finally

Sincere thanks to Andrew Lamb (@alamb) for clear guidance, answering my newbie questions with patience and kind review, which makes the whole arrow-rs (and DataFusion) community friendly and warm. Also, thanks to Adrian for his review and previous work on folding, after all, the review bandwidth of these reviewers is the most limited and precious resource for every open-source project. Besides, Adrian’s detailed article introducing Bloom Filter equipped me with some prerequisite knowledge of Bloom Filter.

## Reference

- Issue #9667: [https://github.com/apache/arrow-rs/issues/9667](https://github.com/apache/arrow-rs/issues/9667)
- PR #9877: [https://github.com/apache/arrow-rs/pull/9877](https://github.com/apache/arrow-rs/pull/9877)
- PR #9969: [https://github.com/apache/arrow-rs/pull/9969](https://github.com/apache/arrow-rs/pull/9969)
- PR #9628 (folding): [https://github.com/apache/arrow-rs/pull/9628](https://github.com/apache/arrow-rs/pull/9628)
- Folding review thread: [https://github.com/apache/arrow-rs/pull/9628#discussion_r3022040191](https://github.com/apache/arrow-rs/pull/9628#discussion_r3022040191)
- Prof. Pavlo’s course - **#09 - Vector Indexes, Inverted Indexes, Filters, Tries ✸ TiDB Talk (CMU Intro to Database Systems)** : [https://www.youtube.com/watch?v=foMMYyycRgk&list=PLSE8ODhjZXjYDBpQnSymaectKjxCy6BYq&index=10](https://www.youtube.com/watch?v=foMMYyycRgk&list=PLSE8ODhjZXjYDBpQnSymaectKjxCy6BYq&index=10) (3:55 - 12:48)
- Adrian's article on Bloom Filter: [https://pydantic.dev/articles/bloom-filter-folding-parquet-logfire](https://pydantic.dev/articles/bloom-filter-folding-parquet-logfire)
- BF calculator: [https://hur.st/bloomfilter/](https://hur.st/bloomfilter/)

## Disclaimer

If readers find anything wrong, please correct me by sending me an email or leaving your comments.

The whole article was written by myself in the first draft, and I only asked AI to proofread my grammar (it is my personal belief that I still want to keep something coming from myself and have the chance to think, organize, and generate by myself).

