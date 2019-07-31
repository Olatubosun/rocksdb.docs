RocksDB configures a certain amount of main memory as a block cache to accelerate data access. Understanding the efficiency of block cache is very important. The block cache analysis and simulation tools help a user to collect block cache access traces, analyze its access pattern, and evaluate alternative caching policies. 

### Table of Contents
**[Quick Start](#quick-start)**<br>
**[Tracing block cache accesses](#tracing-block-cache-accesses)**<br>
**[Trace Format](#trace-format)**<br>
**[Cache Simulations](#cache-simulations)**<br>
**[Analyzing Block Cache Traces](#analyzing-block-cache-traces)**<br>

# Quick Start
db_bench supports tracing block cache accesses. This section demonstrates how to trace accesses when running db_bench. It also shows how to analyze and evaluate caching policies using the generated trace file. 

Create a database: 
```
./db_bench --benchmarks="fillseq" --key_size=20 --prefix_size=20 --keys_per_prefix=0 --value_size=100 --cache_index_and_filter_blocks --cache_size=1048576 --disable_auto_compactions=1 --disable_wal=1 --compression_type=none --min_level_to_compress=-1 --compression_ratio=1 --num=10000000
```
To trace block cache accesses when running `readrandom` benchmark:
```
./db_bench --benchmarks="readrandom" --use_existing_db --duration=60 --key_size=20 --prefix_size=20 --keys_per_prefix=0 --value_size=100 --cache_index_and_filter_blocks --cache_size=1048576 --disable_auto_compactions=1 --disable_wal=1 --compression_type=none --min_level_to_compress=-1 --compression_ratio=1 --num=10000000 --threads=16 -block_cache_trace_file="/tmp/trace" -block_cache_trace_max_trace_file_size_in_bytes=1073741824 -block_cache_trace_sampling_frequency=1
```
Convert the trace file to human readable format:
```
./block_cache_trace_analyzer -block_cache_trace_path=/tmp/block_trace_test_example -human_readable_trace_file_path=/tmp/human_readable_block_trace_test_example
```
Evaluate alternative caching policies: 
```
bash block_cache_pysim.sh /tmp/human_readable_block_trace_test_example /tmp/sim_results/bench 1 0 30
```
Plot graphs:
```
python block_cache_trace_analyzer_plot.py /tmp/sim_results /tmp/sim_results_graphs
```

# Tracing Block Cache Accesses
RocksDB supports block cache tracing APIs `StartBlockCacheTrace` and `EndBlockCacheTrace`. When tracing starts, RocksDB logs detailed information of block cache accesses into a trace file. A user must specify a trace option and trace file path when start tracing block cache accesses.

A trace option contains `max_trace_file_size` and `sampling_frequency`.
- `max_trace_file_size` specifies the maximum size of the trace file. The tracing stops when the trace file size exceeds the specified `max_trace_file_size`.
- `sampling_frequency` determines how frequent should RocksDB trace an access. RocksDB uses spatial downsampling such that it traces all accesses to sampled blocks. A `sampling_frequency` of 1 means tracing all block cache accesses. A `sampling_frequency` of 100 means tracing all accesses on ~1% blocks. 

An example to start tracing block cache accesses: 
```
Env* env = rocksdb::Env::Default();
EnvOptions env_options;
std::string trace_path = "/tmp/block_trace_test_example"
std::unique_ptr<TraceWriter> trace_writer;
DB* db = nullptr;
std::string db_name = "/tmp/rocksdb"

/*Create the trace file writer*/
NewFileTraceWriter(env, env_options, trace_path, &trace_writer);
DB::Open(options, dbname, &db);

/*Start tracing*/
db->StartBlockCacheTrace(trace_opt, std::move(trace_writer));

/* Your call of RocksDB APIs */

/*End tracing*/
db->EndBlockCacheTrace()
```

# Trace file format

# Cache Simulations
Supported simulators. 
- RocksDB cache simulators.
- Python cache simulators. 

Must first convert the binary trace file into human readable trace file. 


# Analyzing Block Cache Traces
Provides insights into how to improve a caching policy. 




