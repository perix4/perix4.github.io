---
layout: post
title: Beware of the ETS bag
subtitle: Performance penalty of ETS bag deduplication
published: false
date: '2020-02-22'
tags: [elixir, ets, bag]
---

Recently we had a problem on our production environment where one of our microservices started behaving badly, being very slow on startup (syncing phase from Kafka). After few hours of debugging, we found the culprit - it was the slow performance of ETS `bag` when dealing with large number of keys with relatively large number of different values within a key.

So what is ETS `bag` anyways? From [erlang ETS docs](https://erlang.org/doc/man/ets.html):

```markdown
Tables are divided into four different types: set, ordered_set, bag, and duplicate_bag. A set or ordered_set table can only have one object associated with each key. A bag or duplicate_bag table can have many objects associated with each key.
```

So basically, bag can have records like:

```elixir
[
  {:key1, :value1},
  {:key1, :value2}
]
```

, but on the other hand `bag` guarantees uniqueness of the values, so if you reinsert record `{:key1, :value2}` you will still have 2 records in that ets table.

As name suggests `duplicate_bag` allows duplication of the values under the same key:

```elixir
[
  {:key1, :value1}
  {:key1, :value2},
  {:key1, :value2},
  {:key1, :value2}
]
```

Surely, deduplication results in performance penalty, but in what extent?

TLDR, SPOILER: We fixed the issue simply by using `duplicate_bag` (and doing deduplication logic manually because dedup of map is EXPENSIVE).

For this I have created the simple program that will prefill the :ets tables (`bag`, and `duplicate_bag`) so we can do benchmarks on that test data, something like:

```elixir
defmodule Bootstrap do
  @moduledoc false

  @size 1_000_000
  @number_of_keys 10_000

  def start do
    init_tables()
    fill_tables()
    save_tables()
  end

  defp init_tables do
    :bag = :ets.new(:bag, [:bag, :named_table, :public])
    :dbag = :ets.new(:dbag, [:duplicate_bag, :named_table, :public])
  end

  def fill_tables do
    1..@size
    |> Enum.map(fn i ->
      key = :rand.uniform(@number_of_keys)
      :ets.insert(:bag, {key, i})
      :ets.insert(:dbag, {key, i})
    end)
  end

  def save_tables do
    :ok = :ets.tab2file(:bag, "test_tables/bag" |> String.to_charlist())
    :ok = :ets.tab2file(:dbag, "test_tables/dbag" |> String.to_charlist())
  end
end

Bootstrap.start()
```

So, our initial data set is 1M rows with 10K distinct keys, so we are expecting around 100 different values per key.

Lets do the bench:

```
Operating System: macOS
CPU Information: Intel(R) Core(TM) i5-7267U CPU @ 3.10GHz
Number of Available Cores: 4
Available memory: 8 GB
Elixir 1.9.4
Erlang 22.2.1

Benchmark suite executing with the following configuration:
warmup: 2 s
time: 10 s
memory time: 0 ns
parallel: 4
inputs: none specified
Estimated total run time: 24 s

Benchmarking bag...
Benchmarking duplicate_bag...

Name                ips    average  deviation  median  99th %
duplicate_bag  206.02 K    4.85 μs  ±3210.37%    1 μs   30 μs
bag              8.91 K  112.18 μs    ±92.72%  105 μs  218 μs

Comparison:
duplicate_bag      206.02 K
bag                  8.91 K - 23.11x slower +107.33 μs
```

As noticed, insert into a `bag` was **23 times** slower than in a duplicate bag. Wow, this is huge penalty.

After this test I was wandering does it get worse with more values under the same keys? Number of initial records is still 1M, benched with `1k` unique keys and `1M` (all unique keys for values).

Benchmark showed this:

```
Operating System: macOS
CPU Information: Intel(R) Core(TM) i5-7267U CPU @ 3.10GHz
Number of Available Cores: 4
Available memory: 8 GB
Elixir 1.9.4
Erlang 22.2.1

Benchmark suite executing with the following configuration:
warmup: 2 s
time: 30 s
memory time: 0 ns
parallel: 4
inputs: none specified
Estimated total run time: 1.07 min

Benchmarking bag_1k...
Benchmarking bag_uniq...

Name         ips    average  deviation   median  99th %
bag_uniq  4.96 K  201.50 μs    ±65.07%   185 μs  393 μs
bag_1k    4.44 K  224.98 μs   ±144.82%   212 μs  474 μs

Comparison:
bag_uniq  4.96 K
bag_1k    4.44 K - 1.12x slower +23.48 μs
```
&nbsp;

#### Conclusion
When working with large number of records in ETS `bag` (million scale), performance penalty on inserts is huge comparing `duplicate_bag` caused by deduplication. Performance also slightly degrades with the increase of the values under the same key in `bag`.