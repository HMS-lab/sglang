# Ascend MemCache as L3 KV Cache

This document explains how to use **Ascend MemCache** as the L3 KV Cache backend for **SGLang HiCache**.

Related documentation:

- [Ascend MemCache Introduction](https://gitcode.com/Ascend/memcache/blob/master/README.md)
- [Ascend MemCache Config Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/memcache_config.md)
- [Ascend MemCache Python API](https://gitcode.com/Ascend/memcache/blob/master/docs/memcache_python_api.md)
- [SGLang HiCache Design](https://docs.sglang.io/advanced_features/hicache_design.html)
- [Ascend MemFabric](https://gitcode.com/Ascend/memfabric_hybrid)
- [Ascend MemCache](https://gitcode.com/Ascend/memcache)

## About MemCache

MemCache is a distributed cache system from Ascend, built on MemFabric underneath, and can provide a high-performance distributed memory pool.
In SGLang HiCache, MemCache can be used as the L3 KV Cache backend to store and reuse KV cache.


## Install Ascend MemCache

[Memcache Official Document](https://gitcode.com/Ascend/memcache/blob/master/docs/install_whl.md)

```bash
pip install memcache_hybrid
```

## Deploy MemCache
### Metaservice
add `metaservice_config.json`
```json
{
    // Meta service start-up url; in K8s meta service master-standby HA, auto-set to Pod IP at startup
    "meta_service_url": "tcp://127.0.0.1:5000",

    // Config store url; in K8s, auto-set to Pod IP at startup
    "config_store_url": "tcp://127.0.0.1:6000",

    // HTTP metrics service url
    "metrics_url": "http://127.0.0.1:8000",

    // Log level: debug, info, warn, error
    "log_level": "info"
}
```

Pass MetaService options via `metaservice_config.json` (see above). Keys below match `memcache_hybrid.MetaConfig` field names.

| Key | Type | Required | Default | Valid range | Description |
| --- | --- | --- | --- | --- | --- |
| `meta_service_url` | string | optional | `tcp://127.0.0.1:5000` | `tcp://<ip>:<port>` | Meta service listen address. Port in [1025, 65535]. |
| `config_store_url` | string | optional | `tcp://127.0.0.1:6000` | `tcp://<ip>:<port>` | Config store address. Port in [1025, 65535]. |
| `metrics_url` | string | optional | `http://127.0.0.1:8000` | `http://<ip>:<port>` | HTTP metrics endpoint. Port in [1025, 65535]. |
| `ha_enable` | boolean | optional | `false` | `true` / `false` | Enable MetaService master/backup HA in a K8s cluster. |
| `log_level` | string | optional | `info` | `debug` / `info` / `warn` / `error` | Log level. |
| `log_path` | string | optional | `/var/log/memcache_hybrid` | relative or absolute path | Log directory. Absolute paths start with `/`. |
| `log_rotation_file_size` | integer | optional | `20` | [1, 500] | Log rotation file size in MB. |
| `log_rotation_file_count` | integer | optional | `50` | [1, 50] | Number of rotated log files to keep. |
| `evict_threshold_high` | integer | optional | `90` | [1, 99] | Eviction high-water mark (%). Max is 99. Eviction is skipped when a single put exceeds 1% of capacity. |
| `evict_threshold_low` | integer | optional | `80` | [0, 98] | Eviction low-water mark (%) after eviction completes. |

For more options, see [MemCache Configuration Guide — MetaService Config](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md#metaservice-config).


## Quick Start npu_memcache as L3 backend

### Shell 1: Start Meta service

```bash
python -m sglang.srt.mem_cache.storage.npu_memcache.start_meta_service --config_path "${metaservice_config_path}"
```

### Shell 2: Start SGLang Server


```bash
python -m sglang.launch_server \
  --model-path ${model_path} \
  --hicache-io-backend kernel_ascend \
  --attention-backend ascend \
  --enable-hierarchical-cache \
  --hicache-storage-backend npu_memcache \
  --hicache-mem-layout page_first_kv_split \
  --hicache-storage-backend-extra-config '{"meta_service_url":"tcp://127.0.0.1:5000", "config_store_url":"tcp://127.0.0.1:6000", "log_level":"info", "world_size":256, "protocol": "device_sdma", "dram_size": "1GB"}'
```

Pass LocalService options via `--hicache-storage-backend-extra-config` (JSON). Keys below match `memcache_hybrid.LocalConfig` field names.

| Key | Type | Required | Default | Valid range | Description |
| --- | --- | --- | --- | --- | --- |
| `meta_service_url` | string | optional | `tcp://127.0.0.1:5000` | `tcp://<ip>:<port>` | Meta service address. Port in [1025, 65535]. In HA, `<ip>` is the cluster IP. |
| `config_store_url` | string | optional | `tcp://127.0.0.1:6000` | `tcp://<ip>:<port>` | Config store address. Port in [1025, 65535]. |
| `log_level` | string | optional | `info` | `debug` / `info` / `warn` / `error` | Log level. |
| `world_size` | integer | optional | `256` | [1, 1024] | Max rank count. Cannot change after ranks connect; restart Meta to update. |
| `protocol` | string | **required** | `host_rdma` | `host_rdma`, `host_urma`, `host_tcp`, `host_shm`, `device_sdma`, `device_rdma` | Transport protocol. `host_shm` requires `dram_size` > 0, `hbm_size` = 0, and no hcom. |
| `hcom_url` | string | optional | `tcp://127.0.0.1:7000` | `tcp://<ip>:<port>` | HCOM address for the DRAM pool. Port in [1024, 65535]. |
| `dram_size` | string / integer | **required** | `1GB` | [0, 1TB] | DRAM pool size. Accepts `134217728`, `2048KB`, `200mb`, `2.5G`, `1TB`, etc. Auto-aligned to 2MB (`host_rdma` / `host_tcp` / `host_shm`) or 1GB (`device_sdma` / `device_rdma`). |
| `hbm_size` | string / integer | optional | `0` | [0, 1TB] | HBM pool size (same format as `dram_size`). Must be `0` when using `host_shm`. |
| `max_dram_size` | string / integer | optional | `64GB` | [0, 1TB] | Max `dram_size` across all local processes. |
| `max_hbm_size` | string / integer | optional | `0` | [0, 1TB] | Max `hbm_size` across all local processes. |


For more options, see [MemCache Configuration Guide — LocalService Config](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md#localservice-config).

## Optional parent-aware writes

Parent metadata is disabled by default and it then doesn't change the way the programme
is used at all (with respect to the previous version). To enable it, add both entries to the
existing `--hicache-storage-backend-extra-config` JSON or configuration file:

```json
{
  "hicache_storage_pass_prefix_keys": true,
  "enable_parent_keys": true
}
```

Keep the other connection and LocalService settings in the same configuration.
`enable_parent_keys` is an SGLang adapter option, not a MemCache `LocalConfig`
field. It must be a boolean. The client and MetaService must support
`batch_put_from_with_parents`. Older clients remain supported when this option is
disabled.

From the point of view of the programme, the method does the same thing as the
defult `_put_batch_zero_copy_impl`, but Memcache uses addtional provided information
to preserve information about prefix of a batch and enable prefix-aware eviction.

For parent-aware v1 and v2 writes, the adapter groups all pages into one path per
component. It passes only the parent of the first node in each path, not a parent
for every object. For example, with preceding page `A` and current pages `B, C`,
MHA rank 0 sends:

```text
keys:         [B_0_k, C_0_k, B_0_v, C_0_v]
parent_keys:  [A_0_k, A_0_v]
buffer_ptrs:  [B_0_k_ptr, C_0_k_ptr, B_0_v_ptr, C_0_v_ptr]
buffer_sizes: [B_0_k_size, C_0_k_size, B_0_v_size, C_0_v_size]
```

Keys are component-major: the complete `k` path comes before the complete `v`
path. Each path keeps logical page order (`B` then `C`), and its entry in
`parent_keys` identifies the corresponding component of `A`. Buffer pointers and
sizes are reordered together with the keys.

Tags and TP/PP/CP suffixes are preserved. At the sequence root, every path has the
empty parent key `""`. Missing prefix context (`prefix_keys=None`) is an error in
this mode, not an implicit sequence root. The same grouping applies to MLA
components and the supported v2 pools (Indexer, Mamba and Draft), including
tail-only transfers.

Writes delegate directly to
`batch_put_from_with_parents(keys, parent_keys, buffer_ptrs, buffer_sizes)`.
The method must return a `List[int]` of length `len(keys)`, in the same
component-major order as the request, not one status per path or a single batch
status. Return `0` for success, including already existing or successfully
skipped objects, and a nonzero code (normally negative) for an error.

The adapter restores page-major status order before combining component statuses
into page-level results: a page succeeds only if all its component statuses are
`0`. For the example above, `[0, 0, -7, 0]` means page `B` failed because `B_0_v`
failed, while page `C` succeeded. MemCache owns allocation, copying and write
completion.
