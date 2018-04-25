### What is a Bloom Filter?
For any arbitrary set of keys, an algorithm may be applied to create a bit array called a Bloom filter. Given an arbitrary key, this bit array may be used to determine if the key *may exist* or *definitely does not exist* in the key set. For a more detailed explanation of how Bloom filters work, see this [Wikipedia article](http://en.wikipedia.org/wiki/Bloom_filter).

In RocksDB, when the filter policy is set every newly created SST file will contain a Bloom filter, which is used to determine if the file may contain the key we're looking for. The filter is essentially a bit array. Multiple hash functions are applied to the given key, each specifying a bit in the array that will be set to 1. At read time also the same hash functions are applied on the search key, the bits are checked, i.e., probe, and the key is definitely does not exists if at least one of the probes return 0.

### Life Cycle
In RocksDB, each SST file has a corresponding Bloom filter. It is created when the SST file is written to storage, and is stored as part of the associated SST file. Bloom filters are constructed for files in all levels in the same way (Note blooms of the last level could be optionally skipped by setting `optimize_filters_for_hits`).

Bloom filters may only be created from a set of keys - there is no operation to combine Bloom filters. When we combine two SST files, a new Bloom filter is created from the keys of the new file. 

When we open an SST file, the corresponding Bloom filter is also opened and loaded in memory. When the SST file is closed, the Bloom filter is removed from memory. To otherwise cache the Bloom filter in block cache, use: `BlockBasedTableOptions::cache_index_and_filter_blocks=true,`.

### Block-based Bloom Filter (old format)

A Bloom filter may only be built if all keys fit in memory. On the other hand, sharding a bloom filter will not affect its false positive rate. Therefore, to alleviate the memory pressure when creating the SST file, in the old format a separate bloom filter is created per each 2KB block of key values.
Details for the format can be found [here](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format#filter-meta-block). At the end an array of the offsets of individual bloom blocks are stored in SST file.

At read time, the offset of the block that might contain the key/value is obtained from the SST index. Based on the offset the corresponding bloom filter is then loaded. If the filter suggests that the key might exist, then it searches the actual data block for the key.

### Full Filters (new format)
The individual filter blocks in the old format are not cache aligned and could result into a lot of cache misses during lookup. Moreover although negative responses (i.e., key does not exist) of the filter saves the search from the data block, the index is already loaded and looked into. The new format, full filter, addresses these issues by creating one filter for the entire SST file. The tradeoff is that more memory is required to cache a hash of every key in the file (versus only keys of a 2KB block in the old format).

Full filter limits the probe bits for a key to be all within the same CPU cache line. This ensures fast lookups by limiting the CPU cache misses to one per key. Note that this is essentially sharding the bloom space and will not affect the false positive rate of the bloom as long as we have many keys. Refer to "The Math" section below for more details.

At read time, RocksDB uses the same format that was used when creating the SST file. Users can specify the format for the newly created SST files by setting `filter_policy` in options. The helper function `NewBloomFilterPolicy` can be used to create both old block-based (the default) and new full filter.

```
extern const FilterPolicy* NewBloomFilterPolicy(int bits_per_key,
    bool use_block_based_builder = true);
}
```
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

Pay attention to these the confusing scenarios:
1. If both whole_key_filtering and prefix are set, prefix are not checked during point lookups.
2. If only the prefix is set, the total number of times prefix bloom is checked is the sum of the stats of point lookup and seeks. Due to absence of true positive stats in seeks, we then cannot have the total false positive rate: only that of of point lookups.

### Customize your own FilterPolicy
FilterPolicy (include/rocksdb/filter_policy.h) can be extended to defined custom filters. The two main functions to implement are:

    FilterBitsBuilder* GetFilterBitsBuilder()
    FilterBitsReader* GetFilterBitsReader(const Slice& contents)
 
In this way, the new filter policy would function as a factory for FilterBitsBuilder and FilterBitsReader. FilterBitsBuilder provides interface for key storage and filter generation and FilterBitsReader provides interface to check if a key may exist in filter.

Notice: This two new interface just works for new filter format. Original filter format still use original method to customize.

### Partitioned Bloom Filter

It is a storage format for full filters. It partitions the filter block into multiple smaller blocks to alleviate the load on block cache.
Read [here](https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters).

### The math

Here is the math to compute the false positive rate (FPR) of bloom filters.
- `m` bits split into `s` shards each of size `m/s`.
- `n` keys
- `k` probe/key
- For each key a shard is selected randomly, and `k` bits are set randomly within the shard.
- After inserting a key, the probability that a particular bit is still 0 is that all the previous keys are either set on other shard `prob = (s-1)/s` or set other bits in the same shard with probability `1 - 1/(m/s)`.
   * prob_0 = `(s-1)/s) + 1/s (1-s/m)) ^ k`
- Using binomial approximation of `(1+x)^k ~= 1 + xk` if `|xk| << 1` and `|x| < 1` we have
   * prob_0 = `1-1/s + 1/s (1-sk/m)` = `1 - k/m` = `(1-1/m)^k`
- Note that with the approximation prob_0(s=1) is equal to prob_0.
- Probability of a bit remained 0 after inserting `n` keys is
   * prob_0_n = `(1-1/m)^nk`
- So the false positive probability is that all of the `k` bits are 1:
   * FPR = `(1 - prob_0_n) ^ k` = `(1- (1-1/m)^kn) ^ k`

Note that the FPR rate is independent of `s`, number of shards. In other words sharding does not affect the FPR as long as `s k/m << 1`. For typical values of `s=512` and `k=6`, 10 bits per key, this is satisfied as long as `n << 307 key`. In full blooms, each shard is of CPU cache line size to reduce cpu cache misses during next probes. `m` will be set `n * bits_per_key + epsilon` to ensure that it is a multiple of the shard size, i.e., the cpu cache line size.
