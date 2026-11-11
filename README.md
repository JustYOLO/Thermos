# Thermos

**Thermos: Keeping Hot Keys in Memory Across Flushes in LSM-Trees**

This repository is the artifact for the Thermos paper. Thermos is an
in-memory absorption layer, implemented in RocksDB, that eliminates hot-key
garbage before it reaches storage.

Thermos is off by default (`enable_hot_table = false`). With it off, the
engine behaves like the RocksDB version it is built on.

## Building

```bash
make -j$(nproc) db_bench
```

---

## Running Thermos

### The three systems in the paper

All three use the same write-buffer memory: four 64 MB memtable slots. In
Thermos, the hottable takes one of the four.

| System | db_bench flags |
|---|---|
| RocksDB | `--enable_hot_table=false` |
| RocksDB in-place mode | `--enable_hot_table=false --inplace_update_support=true --allow_concurrent_memtable_write=false` |
| Thermos | `--enable_hot_table=true --hot_table_write_buffer_size=67108864` |

### Example: Zipfian write-only workload (paper Figure 9)

```bash
./db_bench \
  --benchmarks=fillzip,stats --use_existing_db=false \
  --num=100000000 --zipf_const=0.99 \
  --key_size=16 --value_size=1024 \
  --write_buffer_size=67108864 --max_write_buffer_number=4 \
  --max_total_wal_size=1073741824 \
  --use_direct_reads=true --use_direct_io_for_flush_and_compaction=true \
  --compression_type=none --statistics=1 --seed=912912 \
  --enable_hot_table=true \
  --hot_table_write_buffer_size=67108864 \
  --virtual_flush_interval_flushes=5 --hot_table_rebuild_cost=1.0
```

Swap in the RocksDB or in-place flags from the table above for the
baselines. Flush and compaction volume appear in the `rocksdb.stats` output.
The absorption ratio is `rocksdb.hot.table.write.hit.count` divided by all
writes.

## Reproducing the paper's experiments

The paper's setup (Section V-A) used two Intel Xeon Gold 6338 CPUs, 504 GB of
DRAM and a 2 TB Samsung 870 EVO SATA SSD. Every run shares these flags:

```
--use_direct_reads=true --use_direct_io_for_flush_and_compaction=true
--compression_type=none --statistics=1 --seed=912912
--write_buffer_size=67108864 --max_write_buffer_number=4
--max_total_wal_size=1073741824
```

Synthetic workloads use 16 B keys and 1 KB values. Thermos uses E = 5,
kappa = 1, gamma = 0.5 and p = 0.25 (the defaults) for every workload.

| Paper | Workload | db_bench |
|---|---|---|
| Fig. 5, 8 | Hotspot sweep over update distance d | `--benchmarks=fillhotcold --num=30000000 --hotcold_random=true --hotcold_hot_fraction=0.5 --hotcold_hot_keys=H`, where H = 3114 ... 124560 (d = 2H / 62280) |
| Fig. 9, 10, 11 | Zipfian (0.99) writes, 100M | `--benchmarks=fillzip --num=100000000`; add `--report_flush_time_breakdown=true` for Fig. 10 |
| Table II | Uniform writes, 100M | `--benchmarks=fillrandom --num=100000000` |
| Fig. 12(a) | YCSB A, B, C, D, F | load with `fillzip --num=100000000 --key_range=100000000`, then `ycsbwkld{a,b,c,d,f} --use_existing_db=true` |
| Fig. 12(b) | Mixgraph (Prefix_dist), four read:write mixes | load 50M keys, then `--benchmarks=mixgraph` with the Prefix_dist parameters of Cao et al. (FAST '20) and `--mix_get_ratio` / `--mix_put_ratio` = 0.7/0.3, 0.5/0.5, 0.3/0.7, 0.0/1.0 |
| Fig. 12(c) | Twitter clusters 40 and 41 | `--benchmarks=twittertrace --twitter_trace_file=<cluster trace CSV>` |
| Fig. 13 | Hot key shifts every 10M writes | `fillzip --num=100000000 --key_range=100000000 --hot_key_shift_ops=10000000 --hot_table_window_ops=100000`; for "rebuilds disabled" add `--hot_table_rebuild_on_prediction=false --hot_table_disable_on_low_absorption=false --hot_table_track_hot_keys=false` |
| Fig. 15 | 1-12 writer threads | Zipfian `fillzip` with `--threads=T` and `--num=100000000/T`, with `--value_size=1024` and `--value_size=100` |

Notes:

- The Twitter traces come from Yang et al., "A large-scale analysis of
  hundreds of in-memory key-value cache clusters at Twitter" (TOS 2021). They
  are not included in this repository.
- The HotRAP and DumpKV results (Fig. 14) come from Thermos integrated into
  those code bases, which are not part of this repository.

---

## Statistics

Enable `options.statistics` (or `--statistics=1`) to see these.

| Ticker | Meaning |
|---|---|
| `rocksdb.hot.table.write.hit.count` / `.write.miss.count` | Writes absorbed in place / writes appended to the memtable while a hottable was active |
| `rocksdb.hot.table.read.hit.count` / `.read.miss.count` | Point lookups answered / not answered by the hottable |
| `rocksdb.hot.table.router.match` / `.filtered` / `.false.positives` | Bloom filter outcomes |
| `rocksdb.hot.table.decision.{enable,rebuild,disable,continue}` | Diagnostic engine decisions, one per epoch |
| `rocksdb.hot.table.virtual.flush.count` | Epochs evaluated (virtual flushes) |
| `rocksdb.hot.table.stage.count` / `.pair.activation.count` | Successor hottables built / activated at a memtable switch |
| `rocksdb.hot.table.seal.deferred.count` | Seals put off because they would have stalled writes |
| `rocksdb.hot.table.temp.memtable.count` | Memtables sealed with no hottable active |
| `rocksdb.hot.table.growth.bytes` | Bytes hottables allocated for values that outgrew their slots |
| `rocksdb.flush.key.drop.new` / `.drop.obsolete` | Versions discarded by flush GC |
| `rocksdb.memtable.flush.count` | Memtable flushes completed |

With `report_flush_time_breakdown`, these histograms record flush time by
phase: `rocksdb.flush.mem.iterator.setup.micros`, `.read.merge.micros`,
`.write.block.micros`, `.finish.micros`, `.install.micros`, and
`.hot.key.detect.micros`. The last one is the garbage counting that Thermos
adds to flush.

`FlushJobInfo` (passed to `EventListener::OnFlushCompleted`) also reports
`flush_input_records`, `flush_output_records` and `flush_dropped_*_records`.

---

## License

Thermos keeps RocksDB's dual license: GPLv2 ([COPYING](COPYING)) and Apache
2.0 ([LICENSE.Apache](LICENSE.Apache)), at your option. Code derived from
LevelDB is also covered by [LICENSE.leveldb](LICENSE.leveldb).