# EasyTier Quiet Mode (quiet-v2.6.4) 优化记录

此分支 (`quiet-v2.6.4`) 专门为减少闲置状态下的背景流量而创建。为了达到极致的“静默”效果，在保证组网不掉线的前提下，对源码中的高频保活、测速和路由同步逻辑进行了大幅度的间隔延长。

## 修改总览对比表

| 项目 | 文件 | 原始值 (v2.6.4) | 我的修改 | 后续修正 (2026-09-19) | 说明 |
|---|---|---|---|---|---|
| P2P Ping 基准间隔 | `easytier/src/peers/peer_conn_ping.rs` | 1 秒 | **60 秒** | **30 秒** | 闲时心跳约降 30 倍 |
| Ping 最大退避乘数 | `easytier/src/peers/peer_conn_ping.rs` | 5 (2^5=32s) | **6** (2^6) | **2** (30×2²=120s) | 空闲时最长 120s 一个包，NAT 映射不过期，重连零冷启动；流量仍比原始版低约 3.7 倍 |
| 断线判定丢包阈值 | `easytier/src/peers/peer_conn_ping.rs` | 5 次 | **10 次** | — | 最坏黑窗约 10~11 分钟 |
| OSPF 会话循环休眠 | `easytier/src/peers/peer_ospf_route.rs` | 1 秒 | **60 秒** | — | 事件驱动的即时同步不受影响 |
| OSPF 主动对账间隔 | `easytier/src/peers/peer_ospf_route.rs` | 10 秒 | **60 秒** | — | 路由老化 3660s，远大于此值 |
| 外部网段广播间隔 | `easytier/src/common/constants.rs` | 10 秒 | **60 秒** | — | 降低 Proxy CIDR 广播频率 |
| 代理连接 hedge 间隔 | `easytier/src/gateway/kcp_proxy.rs`、`quic_proxy.rs` | 200 毫秒 | **2000 毫秒** | — | v2.6.4 新增的对冲连接，疑似触发公司 DDoS 告警的主因之一 |
| TCP Keepalive (代理) | `easytier/src/gateway/tcp_proxy.rs` | 5s 首次 / 2s 间隔 | **120s / 30s** | — | 死连接检测从 ~9s 变 ~3min |
| Web 客户端心跳 | `easytier/src/web_client/session.rs` | 1 秒 | 300 秒 | **60 秒** | 300s 会触发服务端会话超时被反复杀掉（见第 5 节） |
| Web 客户端断线重连 | `easytier/src/web_client/mod.rs` | 无等待（紧密循环） | **+1 秒等待** | — | 消除重连风暴 |
| **Web 服务端会话空闲超时** | `easytier-web/src/client_manager/session.rs` | 30 秒 | 未改动 | **120 秒** | 与 60s 心跳配套（2 倍冗余），修复“0 客户端在线”问题 |
| Web 前端轮询间隔 | `easytier-web/frontend` 3 个组件 | 1000 毫秒 | **5000 毫秒** | — | 管理页面数据稍滞后 |
| 退出时任务清理 | `easytier/src/peers/peer_manager.rs` | 无（任务泄漏） | **close_peer + abort_all** | — | 修复断开连接后的任务泄漏 |
| CI 自动构建 | `.github/workflows/core.yml` | 不含本分支 | **加入 quiet-v2.6.4** | — | push 即出构建产物 |

> 未采纳的待定建议（见评估记录）：断线阈值回落到 5 次以缩短黑窗、HedgeExt 错误分支增加退避、`close_peer` 增加超时保护。

## 核心修改清单

### 1. P2P 测速心跳与容错判定 (peer_conn_ping.rs)
原本 EasyTier 默认每秒互相发送 124 Bytes 的 Ping 测速包以测量延迟，这在闲置状态下是最大的流量来源。
- **Ping 基准间隔 (interval)**: 从 `1秒` 修改为 `60秒`。
- **最大指数退避乘数 (max_backoff_idx)**: 从 `5` 修改为 `6`。这意味着在完全闲置时，测速包的发送间隔最长可退避至 $60 \times 2^6 = 3840$ 秒。
- **断线判定阈值 (loss_counter)**: 从 `丢失 5 次` 修改为 `丢失 10 次`。由于 Ping 间隔大幅拉长，为了避免偶发网络波动导致误判断开，容忍度翻倍。最极端情况下需连续 600 秒无响应才会断开该 Peer。

### 2. OSPF 路由事件同步 (peer_ospf_route.rs)
负责全网路由表收敛的守护任务。
- **主事件循环休眠 (session_task sleep)**: 从 `1秒` 修改为 `60秒`。防止频繁强制唤醒带来的不必要状态校验与小包发送。
- **主动对账判定 (sync_as_initiator)**: 从 `10秒` 修改为 `60秒`。降低发起方强制推送路由表的频率。
- *(注：路由的老化时间硬编码为 3660 秒，此修改不会导致路由过期。)*

### 3. 代理网段/外部网络同步 (constants.rs)
- **OSPF_UPDATE_MY_GLOBAL_FOREIGN_NETWORK_INTERVAL_SEC**: 从 `10秒` 修改为 `60秒`。降低开启 Proxy CIDR 时的广播频率。

### 4. 传输层心跳 (TCP Keepalive & Web Client)
- **TCP Keepalive**: 已修改为 120 秒首次 / 30 秒间隔，减少底层 TCP 连接保活包。
- **Web Session Heartbeat**: 已修改为 60 秒（原 300 秒会导致会话被服务端反复杀掉，见第 5 节），降低节点与中心配置服务器的通信频率。

### 5. Web 会话心跳与服务端超时的配套 (2026-08-17 修复)
easytier-web 服务端对每个节点会话设有 RPC 空闲超时（`easytier-web/src/client_manager/session.rs` 的 `set_rx_timeout`，上游默认 30 秒）：30 秒收不到节点的任何 RPC 包就销毁会话。心跳间隔改为 300 秒后，每个会话都会在心跳后 30 秒被服务端杀死，节点随后重连，形成约 45 秒周期的"上线 30 秒 / 离线十几秒"循环，表现为管理页面经常显示 0 客户端在线，并产生大量重连流量。
- **客户端心跳** (`easytier/src/web_client/session.rs`): 300 秒 → **60 秒**。
- **服务端会话空闲超时** (`easytier-web/src/client_manager/session.rs`): 30 秒 → **120 秒**（为心跳间隔的 2 倍冗余）。

## 效果总结
经过以上修改，无业务流量状态下的 EasyTier 会从“每秒数个控制包”进入深度静默状态。常规的心跳和对账频率被压制在 1~5 分钟级别，闲置流量开销降低 95% 以上，且断网发现与恢复时间依然保持在合理范围（约 5 分钟）。
