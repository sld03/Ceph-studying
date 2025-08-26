# Ceph df

# 1. Calculate by pool

```cpp
void PGMapDigest::dump_pool_stats_full(
  const OSDMap &osd_map,
  stringstream *ss,
  ceph::Formatter *f,
  bool verbose) const
{
```

Hàm này sẽ gọi đến hàm bên dưới, có nhiệm vụ lấy ra các thông tin về available space.

```cpp
void PGMapDigest::dump_object_stat_sum(
  TextTable &tbl, ceph::Formatter *f,
  const pool_stat_t &pool_stat, uint64_t avail,
  float raw_used_rate, bool verbose, bool per_pool, bool per_pool_omap,
  const pg_pool_t *pool)
{
```

Hàm trên in ra màn hình các giá trị statistic của từng pool theo các tham số

```cpp
stored_normalized = stored_data_normalized + stored_omap_normalized
```