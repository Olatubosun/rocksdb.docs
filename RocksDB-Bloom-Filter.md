### What is a Bloom Filter?
For any arbitrary set of keys, an algorithm may be applied to create a bit array called a Bloom filter. Given an arbitrary key, this bit array may be used to determine if the key *may exist* or *definitely does not exist* in the key set. For a more detailed explanation of how Bloom filters work, see this [Wikipedia article](http://en.wikipedia.org/wiki/Bloom_filter).

In RocksDB, when the filter policy is set, every newly created SST file will contain a Bloom filter, which is used to determine if the file may contain the key we're looking for. The filter is essentially a bit array. Multiple hash functions are applied to the given key, each specifying a bit in the array that will be set to 1. At read time also the same hash functions are applied on the search key, the bits are checked, i.e., probe, and the key definitely does not exist if at least one of the probes return 0.

The example of setting up a bloom filter:

```
  rocksdb::BlockBasedTableOptions table_options;
  table_options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10, false));
  my_cf_options.table_factory.reset(
      rocksdb::NewBlockBasedTableFactory(table_options));
```

### Life Cycle
When configured in RocksDB, each SST file is created with a Bloom filter, embedded in the SST file itself. Bloom filters are generally constructed for files in all levels in the same way, except that the last level can be skipped by setting `optimize_filters_for_hits`, and overriding `FilterPolicy::GetBuilderWithContext` [offers customizability](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter#customize-your-own-filterpolicy).

Because of different sizes and other Bloom filter limitations, SST Bloom filters are not generally compatible with each other for optimized AND/OR composition. Even when we combine two SST files, a new Bloom filter is created from scratch with the keys of the new file, in part to ensure predictable bits/key and, thus, false positive rate.

When we open an SST file, the corresponding Bloom filter is also opened and loaded in memory. When the SST file is closed, the Bloom filter is removed from memory. To otherwise cache the Bloom filter in block cache, use: `BlockBasedTableOptions::cache_index_and_filter_blocks=true,`.

### Block-based Bloom Filter (old format)

With this API, a Bloom filter could only be built if all keys fit in memory. On the other hand, sharding keys across bloom filters does not affect the overall false positive rate if each filter is sized for the number of keys added. Therefore, to alleviate the memory pressure when creating the SST file, in the old format a separate bloom filter is created per each 2KB block of key values.
Details for the format can be found [here](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format#filter-meta-block). At the end an array of the offsets of individual bloom blocks is stored in SST file.

At read time, the offset of the block that might contain the key/value is obtained from the SST index. Based on the offset the corresponding bloom filter is then loaded. If the filter suggests that the key might exist, then it searches the actual data block for the key.

### Full Filters (new format)
The individual filter blocks in the old format are not cache aligned and could result into a lot of cache misses during lookup. Moreover although negative responses (i.e., key does not exist) of the filter saves the search from the data block, the index is already loaded and looked into. The new format, full filter, addresses these issues by creating one filter for the entire SST file. The tradeoff is that more memory is required to cache a hash of every key in the file (versus only keys of a 2KB block in the old format).

Full filter limits the probe bits for a key to be all within the same CPU cache line. This ensures fast lookups and maximizes CPU cache effectiveness by limiting the CPU cache misses to one per key (per filter). Note that this is essentially sharding the bloom space and has only a small effect on the false positive rate as long as there are many keys. Refer to "The Math" section below for more details.

At read time, RocksDB uses the same format that was used when creating the SST file. Users can specify the format for the newly created SST files by setting `filter_policy` in options. The helper function `NewBloomFilterPolicy` can be used to create both old block-based (the default) and new full filter.

```
extern const FilterPolicy* NewBloomFilterPolicy(int bits_per_key,
    bool use_block_based_builder = true);
}
```

The first underlying Bloom filter implementation used for full filters (and partitioned filters) had some flaws, and a new implementation is used with `format_version`=5 (version 6.6) and later. First, the original full filter could not get an FP rate better than about 0.1%, even at 100 bits/key. The new implementation is below 0.1% FP rate with only 16 bits/key. Second, possibly unlikely, the original full filter would have degraded FP rates with millions of keys in a single filter, because of inherent limitations of 32-bit hashing. The new implementation easily scales to many billions of keys in a single filter, thanks to a 64-bit hash. Aside from extra intermediate data during construction (64 bits per key rather than 32), the new implementation is generally faster also.

#### Prefix vs. whole key

By default a hash of every whole key is added to the bloom filter. This can be disabled by setting `BlockBasedTableOptions::whole_key_filtering` to false. When Options.prefix_extractor is set, a hash of the prefix is also added to the bloom. Since there are less unique prefixes than unique whole keys, storing only the prefixes in bloom will result into smaller blooms with the down side of having larger false positive rate. Moreover the prefix blooms can be optionally (using `check_filter` when creating the iterator) also used during `::Seek` and `::SeekForPrev` whereas the whole key blooms are only used for point lookups.

#### Statistic

Here are the statistics that can be used to gain insight of how well your full bloom filter settings are performing in production:

The following stats are updated after each `::Seek` and `::SeekForPrev` if prefix is enabled and `check_filter` is set.
- `rocksdb.bloom.filter.prefix.checked`: seek_negatives + seek_positives
- `rocksdb.bloom.filter.prefix.useful`: seek_negatives

The following stats are updated after each point lookup. If `whole_key_filtering` is set, this is the result of checking the bloom of the whole key, otherwise this is the result of checking the bloom of the prefix.
- `rocksdb.bloom.filter.useful`: [true] negatives
- `rocksdb.bloom.filter.full.positive`: positives
- `rocksdb.bloom.filter.full.true.positive`: true positives

So the false positive rate of point lookups is computed by (positives - true positives) / (positives - true positives + true negatives).

Pay attention to these confusing scenarios:
1. If both whole_key_filtering and prefix are set, prefix are not checked during point lookups.
2. If only the prefix is set, the total number of times prefix bloom is checked is the sum of the stats of point lookup and seeks. Due to absence of true positive stats in seeks, we then cannot have the total false positive rate: only that of of point lookups.

### Customize your own FilterPolicy
FilterPolicy (include/rocksdb/filter_policy.h) can be extended to define custom filters. The two main functions to implement are:

    FilterBitsBuilder* GetFilterBitsBuilder()
    FilterBitsReader* GetFilterBitsReader(const Slice& contents)
 
In this way, the new filter policy would function as a factory for FilterBitsBuilder and FilterBitsReader. FilterBitsBuilder provides interface for key storage and filter generation and FilterBitsReader provides interface to check if a key may exist in filter.

A newer alternative to `GetFilterBitsBuilder()` is also available (override only one of the two):

    FilterBitsBuilder* GetBuilderWithContext(const FilterBuildingContext&)

This variant allows custom filter configuration based on contextual information such as table level, compaction style, and column family. See [`LevelAndStyleCustomFilterPolicy`](https://github.com/facebook/rocksdb/commit/ca3b6c28c90663db58a27c074d0c79c5cf124a73) for an example selecting between configurations of built-in filters.

Notice: This builder/reader interface only works for full and partitioned filters (new format).

### Partitioned Bloom Filter

It is a storage format for full filters. It partitions the filter block into multiple smaller blocks to alleviate the load on block cache.
Read [here](https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters).

### The math

Refer e.g. to [the wikipedia article](https://en.wikipedia.org/wiki/Bloom_filter#Probability_of_false_positives) for standard false positive rate of a Bloom filter.

The CPU cache-friendly Bloom filter variant is presented and analyzed in [this 2007 paper](http://algo2.iti.kit.edu/documents/cacheefficientbloomfilters-jea.pdf)(page 4, formula 3). In short, the Bloom filter is sharded into `s` shards, each of size `m/s`. If `s` were much smaller than `n` (number of keys), each shard would have approximately `n/s` keys mapping to it, and the false positive rate is like a standard Bloom filter using `m/s` for `m` and `n/s` for `n`. This yields essentially the same false positive rate as for a standard Bloom filter with the original `m` and `n`.

For CPU-cache locality of probes, `s` is within a couple orders of magnitude of `n`, which leads to noticeable variance in the number of keys mapped to each shard. (This is the phenomenon behind "clustering" in hashing.) This matters because the false positive rate is non-linear in the number of keys added. Even though the median false positive rate of a shard is the same as a standard Bloom filter, what matters is the mean false positive rate, or the expected value of the false positive rate of a shard.

This can be understood using the [Poisson distribution](https://en.wikipedia.org/wiki/Poisson_distribution) ([as an approximation of a binomial distribution](https://en.wikipedia.org/wiki/Binomial_distribution#Poisson_approximation)), using parameter `n/s`, the expected number of keys mapping to each shard. You can compute the expected/effective false positive rate using a summation based on the pdf of a Poisson distribution (in 2007 paper referenced above), or you can use an approximation I have found to be very good:
- The expected false positive rate is close to the average of the false positive rate for one standard deviation above and below the expected number of keys per shard.

The standard deviation is the square root of that `n/s` parameter. In a standard configuration, `m/s = 512` and `m/n = 10`, so `n/s = 51.2` and `sqrt(n/s) = 7.16`. With `k=6`, we use the standard Bloom filter false positive rate formula, substituting `51.2 +/- 7.16` for `n` and `512` for `m`, yielding 0.43% and 1.48%, which averages to 0.95% for the CPU cache local Bloom filter. The same Bloom filter without cache locality has false positive rate 0.84%.

This modest increase in false positive rate, here a factor of 1.13 higher, is considered quite acceptable for the speed boost, especially in environments heavily utilizing many cores.