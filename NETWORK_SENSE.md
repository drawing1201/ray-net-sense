# Ray Network Sensing (ray-net-sense)

本仓库基于 **Ray 2.53.0**，添加了一个 **Phase-0 级别的“节点间网络探测（Network Sensing）”** 原型：

- **RTT 探测**（并维护窗口统计 + EWMA）
- **带宽探测**（通过 gRPC 发送固定大小 payload 估算吞吐）
- **上报到 GCS Internal KV**（每个 raylet 周期写入自己的出边探测结果）
- **Dashboard 聚合接口**：`GET /api/v0/network_topology`
- **CLI 命令**：`ray network status`

> 说明：本仓库的 `main` 分支是一个 snapshot（单根提交），并且 **不包含** `.github/workflows/*`。

---

## 1. 功能概览

### 1.1 新增 RPC（raylet ↔ raylet）
- `NodeManagerService::NetPing`：用于 RTT 探测
- `NodeManagerService::NetBandwidthProbe`：用于带宽探测（请求里携带 `payload`，reply 回包大小）

相关代码：
- `src/ray/protobuf/node_manager.proto`
- `src/ray/rpc/node_manager/node_manager_server.h`
- `src/ray/raylet_rpc_client/raylet_client_interface.h`
- `src/ray/raylet_rpc_client/raylet_client.cc`

### 1.2 raylet 周期探测 + 上报
raylet 在 `NodeManager` 内部周期执行 `NetworkSenseTick()`：

- 遍历集群内其它节点
- 发起 `NetPing` 并测量 RTT（客户端侧测量）
- 维护 **窗口样本**（`M`）并计算：
  - `rtt_p50_ms`：窗口 p50
  - `jitter_ms`：窗口 `p95 - p50`
  - `rtt_ms`：对 `p50` 做 EWMA（默认 `alpha=0.3`）
- （可选）带宽探测：发送固定大小 `payload`，按 `payload_bytes / elapsed` 估算 `bw_mbps`
  - `bw_p50_mbps`：窗口 p50
  - `bw_mbps`：对 `p50` 做 EWMA
- 将结果写入 **GCS Internal KV**：
  - `namespace = network_sense`
  - `key = edge_samples:<src_node_id_hex>`
  - `value = JSON`（见下文格式）

相关代码：
- `src/ray/raylet/node_manager.cc`
- `src/ray/common/ray_config_def.h`
- `src/ray/common/constants.h`

---

## 2. 数据格式

### 2.1 Internal KV 单节点上报（per-raylet report）
Key：`network_sense/edge_samples:<src_node_id>`

Value（JSON，示例）：
```json
{
  "node_id": "<src_node_id_hex>",
  "ts_ms": 1730000000000,
  "edges": [
    {
      "dst_node_id": "<dst_node_id_hex>",
      "rtt_ms": 0.35,
      "rtt_p50_ms": 0.33,
      "jitter_ms": 0.05,
      "bw_mbps": 950.0,
      "bw_p50_mbps": 980.0
    }
  ]
}
```

注意：
- `rtt_ms` / `bw_mbps` 是 EWMA 结果（更平滑）
- `rtt_p50_ms` / `bw_p50_mbps` 是窗口 p50（更“瞬时”）
- `jitter_ms` 当前定义为窗口 `p95 - p50`

### 2.2 Dashboard 聚合快照
接口：`GET /api/v0/network_topology`

返回是 Ray Dashboard 的标准 envelope，聚合结果在 `result` 字段内：
- `result.nodes`: 节点列表（含 `last_report_ts_ms`）
- `result.edges`: 有向边列表（含 RTT/jitter/bw 字段）

---

## 3. 配置项（RayConfig / system-config）

全部通过 Ray system-config 注入（`ray start --system-config='{"k":...}'` 或 `ray.init(_system_config={...})`）。

RTT：
- `network_sense_enabled`（bool，默认 `false`）：总开关
- `network_sense_tick_ms`（uint64，默认 `2000`）：探测周期（ms）
- `network_sense_rtt_m`（uint32，默认 `5`）：RTT 窗口大小
- `network_sense_rtt_alpha`（double，默认 `0.3`）：EWMA alpha
- `network_sense_rtt_ttl_ms`（uint64，默认 `6000`）：过期时间（ms）
- `network_sense_rtt_concurrency_per_node`（uint32，默认 `2`）：每个 dst 的并发 ping 上限

带宽：
- `network_sense_bw_enabled`（bool，默认 `false`）：带宽探测开关
- `network_sense_bw_payload_bytes`（uint64，默认 `1048576`=1MiB）：探测 payload 大小
- `network_sense_bw_m`（uint32，默认 `3`）：带宽窗口大小
- `network_sense_bw_alpha`（double，默认 `0.3`）：带宽 EWMA alpha
- `network_sense_bw_concurrency_per_node`（uint32，默认 `1`）：每个 dst 的并发带宽探测上限

> 成本提醒：带宽探测是 **O(N^2)** 且会产生额外网络流量，建议先在小集群验证并调大 `tick_ms`/调小 `payload_bytes`。

---

## 4. 如何运行与验证

下面给两种方式：源码安装（裸机）和 Docker（多机更推荐）。

### 4.1 裸机（从源码安装）
1) 安装 Ray（会编译 C++ / `_raylet`，需要 bazel/bazelisk）
```bash
cd python
pip install -r requirements.txt
pip install -e . --verbose
```

2) 启动 head 并打开网络探测
```bash
ray stop || true
ray start --head --dashboard-host=0.0.0.0 \
  --system-config='{"network_sense_enabled":true,"network_sense_bw_enabled":true}'
```

3) worker 加入
```bash
ray start --address=<head_ip>:6379 \
  --system-config='{"network_sense_enabled":true,"network_sense_bw_enabled":true}'
```

4) 验证
```bash
ray status --address <head_ip>:6379
ray network status --address <head_ip>:6379
curl "http://<head_ip>:8265/api/v0/network_topology"
```

### 4.2 Docker（推荐跨多机部署）
**强烈建议**用 `--network=host`，否则跨机器互联复杂，且 RTT/带宽会混入 NAT/bridge 开销。

启动 head（示例）：
```bash
docker run -d --name ray-head \
  --network host --ipc=host --shm-size=20g --ulimit nofile=65536:65536 \
  --gpus all \
  -e RAY_SYS_CFG='{"network_sense_enabled":true,"network_sense_bw_enabled":true}' \
  ray-netsense:2.53.0 \
  bash -lc 'ray stop || true; ray start --head --node-ip-address=<HEAD_IP> --port=6379 --dashboard-host=0.0.0.0 --dashboard-port=8265 --system-config="$RAY_SYS_CFG"; tail -f /dev/null'
```

启动 worker（示例）：
```bash
docker run -d --name ray-worker \
  --network host --ipc=host --shm-size=20g --ulimit nofile=65536:65536 \
  --gpus all \
  -e RAY_SYS_CFG='{"network_sense_enabled":true,"network_sense_bw_enabled":true}' \
  ray-netsense:2.53.0 \
  bash -lc 'ray stop || true; ray start --address=<HEAD_IP>:6379 --node-ip-address=<WORKER_IP> --system-config="$RAY_SYS_CFG"; tail -f /dev/null'
```

验证（在 head）：
```bash
docker exec -it ray-head ray status --address <HEAD_IP>:6379
sleep 10
docker exec -it ray-head ray network status --address <HEAD_IP>:6379
curl "http://<HEAD_IP>:8265/api/v0/network_topology" | python3 -m json.tool
```

---

## 5. 常见问题 / Troubleshooting

### 5.1 `ray network status` 没有任何输出
- 先等 10 秒（默认 tick=2s）
- 确认 head/worker 都启用了 `network_sense_enabled=true`
- Docker 场景请使用 `--network=host`
- 检查防火墙：至少需要 `6379`（GCS）、raylet/node-manager 端口、worker 端口范围等可达

### 5.2 编译时遇到依赖下载 429（GitHub 限流）
- 给 Bazel 配置 `~/.netrc`，并启用 `--repository_cache` 缓存
- 典型做法（不要把 token 提交到仓库/镜像层）：
  - 在构建机上写 `~/.netrc`（`chmod 600 ~/.netrc`）
  - 设置：`export BAZEL_ARGS="--repository_cache=$HOME/bazel-repo-cache"`

### 5.3 编译时 `dl.google.com` 超时（Go SDK 下载失败）
- 这是 Bazel 下载 Go toolchain 的网络问题
- 解决方式之一：在构建镜像时禁用 extra cpp，减少触发 `go_sdk` 的概率：
  - `RAY_DISABLE_EXTRA_CPP=1 pip install -e .`
- 或在能访问 `dl.google.com` 的环境（例如 WSL2）构建镜像后再分发

---

## 6. 开发者入口（代码位置速查）

- RTT/带宽探测逻辑：`src/ray/raylet/node_manager.cc`（`NetworkSenseTick`）
- RPC 定义：`src/ray/protobuf/node_manager.proto`
- Dashboard 接口：`python/ray/dashboard/modules/state/state_head.py`（`/api/v0/network_topology`）
- CLI：`python/ray/scripts/scripts.py`（`ray network status`）
- 配置项：`src/ray/common/ray_config_def.h`
- Internal KV 常量：`src/ray/common/constants.h`
