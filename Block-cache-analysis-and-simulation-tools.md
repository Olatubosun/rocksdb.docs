RocksDB configures a certain amount of main memory as a block cache to accelerate data access. Understanding the efficiency of block cache is very important. The block cache analysis and simulation tools help a user to collect block cache access traces, analyze its access pattern, and evaluate alternative caching policies. 

# Tracing block cache accesses
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
To trace block cache accesses in db_bench. 
```
./db_bench --block_cache_trace_file=/tmp/block_trace_test_example --block_cache_trace_sampling_frequency=100 -block_cache_trace_max_trace_file_size_in_bytes=1024*1024*1024
```

# Cache simulations
Supported simulators. 
- RocksDB cache simulators.
- Python cache simulators. 

Must first convert the binary trace file into human readable trace file. 


# Analyzing block cache traces

# Trace file format


