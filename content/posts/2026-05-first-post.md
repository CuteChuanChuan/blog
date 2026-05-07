+++
title = "Hello, World — Setting Up This Blog"
date = 2026-05-07
description = "First post: verifying the Zola + tabi + Cloudflare Pages stack with code blocks, KaTeX, and a Mermaid diagram."

[taxonomies]
tags = ["zola", "tabi", "meta"]
categories = ["meta"]

[extra]
toc = true
katex = true
mermaid = true
+++

This post exists to verify the full rendering pipeline: code blocks, math,
diagrams, RSS, and search. If you're reading this on the deployed site,
everything works.

## Code blocks

A short Rust snippet:

```rust
fn fibonacci(n: u32) -> u64 {
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 0..n {
        let next = a + b;
        a = b;
        b = next;
    }
    a
}

fn main() {
    for i in 0..10 {
        println!("fib({i}) = {}", fibonacci(i));
    }
}
```

And the SQL I run more often than I'd like to admit:

```sql
SELECT
    table_schema,
    table_name,
    pg_size_pretty(pg_total_relation_size(quote_ident(table_schema) || '.' || quote_ident(table_name))) AS total_size
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(quote_ident(table_schema) || '.' || quote_ident(table_name)) DESC
LIMIT 20;
```

## Math (KaTeX)

The mass-energy equivalence, inline: $E = mc^2$.

A block equation — the Cauchy–Schwarz inequality:

$$
\left( \sum_{i=1}^{n} a_i b_i \right)^2 \le \left( \sum_{i=1}^{n} a_i^2 \right) \left( \sum_{i=1}^{n} b_i^2 \right)
$$

## Diagram (Mermaid)

A typical CDC pipeline:

{% mermaid() %}
flowchart LR
    A[Postgres] -->|Debezium| B[(Kafka)]
    B --> C[Flink]
    C --> D[(Iceberg / Nessie)]
    D --> E[Trino / DuckDB]
{% end %}

## What's next

- Wire up Cloudflare Pages and a custom domain
- Move large images to Cloudflare R2
- Turn on giscus comments
