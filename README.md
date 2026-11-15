# CTP 期权多前置行情池与风控交易一体化平台

> **CTP Option Multi-Front Market Data Pool and Risk-Controlled Trading Platform**

[![Status](https://img.shields.io/badge/status-skeleton%20%2F%20P0-yellow.svg)](#当前状态)
[![Rust](https://img.shields.io/badge/Rust-1.75%2B-orange.svg)](https://www.rust-lang.org/)
[![CTP](https://img.shields.io/badge/CTP-6.7.7-blue.svg)](https://github.com/adminmima/ctp-option-multi-front-market-data-pool-and-risk-controlled-trading-platform)
[![License](https://img.shields.io/badge/license-Apache--2.0-green.svg)](./LICENSE)

---

## 目录

- [这是什么](#这是什么)
- [当前状态](#当前状态)
- [项目从哪来](#项目从哪来)
- [架构](#架构)
- [目录结构](#目录结构)
- [六大 crate 职责](#六大-crate-职责)
- [目录与文件命名约定（重要）](#目录与文件命名约定重要)
- [技术选型](#技术选型)
- [路线图](#路线图)
- [快速开始](#快速开始)
- [配置要点](#配置要点)
- [已知约束与风险](#已知约束与风险)
- [文档索引](#文档索引)
- [贡献与提交规范](#贡献与提交规范)
- [许可与免责](#许可与免责)

---

## 这是什么

一个面向**国内期权市场**的一体化平台，解决三件事：

1. **行情**：把 3 万+ 期权合约分散到多个 CTP 行情前置（多 IP、多站点），单前置断开只迁移该片合约，不触发全局重订阅。
2. **交易**：交易连接独立于行情池，单主前置 + 备前置切换，认证 → 登录 → 结算确认 → 报单/撤单全链路可复用。
3. **风控**：报单**前**同步拦截（名义金额、持仓限额、撤单率、自成交），不是事后告警，不碰柜台。

一句话：**行情要稳、交易要通、风控要拦在报单之前。**

---

## 当前状态

> 更新于 2026-11-10

**阶段：P0 — 骨架已建，代码未写。**

| 项 | 状态 |
|---|---|
| 项目目录与中文命名结构 | ✅ 已建 |
| Workspace 六 crate 骨架 | ✅ 已建 |
| `Cargo.toml` / `.gitignore` / `.cargo/config.toml` | ✅ 已写 |
| `docs_文档/` 七篇占位 | ✅ 已建（空文件） |
| `scripts_脚本/` 五个占位 | ✅ 已建（空文件） |
| 各 crate 的 `lib.rs` / `main.rs` | ⬜ 空文件，`cargo check` 可通过 |
| 长英文名源文件（业务代码） | ⬜ 空文件，未在 `lib.rs` 注册 `mod` |
| 行情池 / 交易执行 / 风控 / 监控规则 | ⬜ 待 P1–P5 实现 |
| 实盘切换 | ⬜ 未开始 |

- **现在能做的**：`cargo check --workspace` 通过，目录和文件结构完整。
- **现在不能做的**：连 CTP、收 tick、报单、风控拦截——这些是 P1 之后的事。

---

## 项目从哪来

由两个已验证的工程合并演进：

| 来源 | 做了什么 | 留下什么 |
|---|---|---|
| `option_monitor` | Rust + CTP 行情，单前置，3 万合约订阅，监控规则 + 运行报告 | `model` / `pipeline` / `monitor` / `run_report` 的逻辑直接平移 |
| `trend_analyzer_ctp_test_...` | 国泰君安 CTP 仿真外部接入，13 项测试，Python→Rust 迁移，ctp2rs 6.7.7 跑通 | `open_session`（认证 + 登录 + 结算确认）、报单/撤单、断线重连、错误码手册 |

**合并时修正的核心问题**：

- 行情侧原用 `ctp-rs 0.4`（编译期静态链接，换 `.so` 版本要重编），交易侧用 `ctp2rs`（运行时 dynlib 加载，已验证 6.7.7 符号对齐）。**统一到 `ctp2rs`**，行情池里不同前置版本不一致也能跑。
- 行情侧 `OnFrontDisconnected` 直接 `break` 退出，**没有重连**。交易侧 `run_test_2_5` 的 `'outer` 重连循环 + `open_session` 复用是正确模式，**抽到行情池**。
- 行情侧 `RUN_TIMEOUT_SECS = 50` 到点强制退出，是测试脚本不是服务。**改为长驻 + 定时报告快照**。

---

## 架构

```
                    ┌──────────────────────────────────────────────┐
                    │  multi_front_connection_pool                 │
                    │  ├─ member[0] → 上海电信站点1  IP组           │
                    │  ├─ member[1] → 上海电信站点2  IP组           │
                    │  ├─ member[2] → 上海联通站点1  IP组           │
                    │  └─ member[N] → ...                           │
                    │  每 member 独立 MdApi 会话 / 独立认证登录      │
                    └──────────────────┬───────────────────────────┘
                                       │ 统一 mpsc (crossbeam-channel)
                                       ▼
                    ┌──────────────────────────────────────────────┐
                    │  Tick 汇聚（platform-app）                     │
                    │  按 InstrumentId 去重 + UpdateTime 取最新      │
                    └────────┬─────────────────────┬───────────────┘
                             │                     │
                             ▼                     ▼
              ┌────────────────────────┐  ┌──────────────────────────┐
              │ monitor-rules          │  │ storage                  │
              │ spread / price / jump  │  │ jsonl / parquet（先）     │
              │ → Alert                │  │ TDengine / DolphinDB（后）│
              └───────────┬────────────┘  └──────────────────────────┘
                          │
                          ▼
                    runtime_report_writer

     ┌────────────────────────────────────────────────────────────┐
     │  trading-executor（独立于行情池，单主前置 + 备前置）         │
     │  session（认证→登录→结算确认）→ insert / cancel / query      │
     └───────────────────────┬────────────────────────────────────┘
                             │ 报单前同步调用
                             ▼
     ┌────────────────────────────────────────────────────────────┐
     │  risk-control-engine                                        │
     │  pre_trade_order_validation                                 │
     │  名义金额 / 持仓限额 / 撤单率 / 自成交                       │
     │  Ok  → req_order_insert                                     │
     │  Err → 本地拒绝，日志 [RISK REJECT]，不碰柜台                │
     └────────────────────────────────────────────────────────────┘
```

**关键设计决策**：

1. **行情池与交易执行器完全解耦**。行情多连接并发；交易单主连接 + 主备切换，**不并发报单**——避免同账户多连接触发流控和持仓同步问题。
2. **风控同步拦截在 `req_order_insert` 之前**。现有 `monitor/*` 是 tick 回调里的事后告警，不能当风控。风控引擎是独立 crate，输入订单，输出 `Ok/Err`。
3. **存储先轻后重**。第一版 tick 落 jsonl/parquet，不引入 TDengine。曲面计算是核心需求时再上 DolphinDB。
4. **合约分片按交易所前缀**。3 万合约不是平均分给 10 个 IP，按上交所 / 上期所 / 郑商所 / 大商所 / 中金所 / 广期所切，每 member 负责 1–2 个交易所，断线只迁移该所订阅。

---

## 目录结构

```
ctp-option-multi-front-...-platform_CTP期权多前置行情池与风控交易一体化平台/
├── Cargo.toml                      # workspace 根
├── Cargo.lock
├── Makefile
├── README.md                       # 本文件
├── LICENSE                         # Apache-2.0
├── PROJECT_LOG.md
├── .gitignore
├── .cargo/config.toml              # 代理配置
├── config_配置/
│   ├── config.example.toml
│   └── config.toml                 # 不入 git
├── crates_代码工程/
│   ├── ctp-option-common-model_公共数据模型/
│   ├── ctp-option-market-data-connector_行情连接器/
│   ├── ctp-option-trading-executor_交易执行器/
│   ├── ctp-option-risk-control-engine_风控引擎/
│   ├── ctp-option-monitor-rules_行情监控规则/
│   └── ctp-option-platform-app_平台主程序/
├── docs_文档/                      # 01–07 七篇
├── scripts_脚本/                   # 初始化 / 编译 / 运行 / 解析
├── data_数据/
│   ├── baseline_基线数据/raw_excel_原始Excel/
│   ├── contracts_合约清单/contracts.txt
│   ├── ticks_行情快照/
│   └── flow_会话流文件/
├── sdk_ctp库/
│   ├── include_头文件/             # ThostFtdc*.h
│   └── lib_动态库/                 # libthostmduserapi_se.so / libthosttraderapi_se.so
├── reports_运行报告/
├── logs_运行日志/
├── evidence_测试证据/
└── storage_持久化存储/
    └── tdengine_schema_时序库建表脚本/
```

---

## 六大 crate 职责

| crate | 职责 | 关键文件 |
|---|---|---|
| `common-model` | 纯数据模型，无 IO，无 CTP 依赖 | `option_contract_identifier` / `market_data_tick_snapshot` / `monitoring_alert_notification` / `trading_order_request` / `platform_error_codes` |
| `market-data-connector` | ctp2rs MdApi 封装、连接池、分片、主备切换 | `ctp_market_data_api_wrapper` / `market_data_connection_member` / `multi_front_connection_pool` / `instrument_sharding_allocator` / `front_connection_failover` |
| `trading-executor` | 认证登录结算、报单、撤单、查询、重连 | `trading_session_authentication` / `order_insert_request_sender` / `order_cancel_request_sender` / `account_position_order_query` / `trading_connection_reconnect` |
| `risk-control-engine` | 报单前同步风控 | `pre_trade_order_validation` / `notional_and_position_limits` / `order_cancel_rate_monitor` / `self_trade_prevention_check` |
| `monitor-rules` | 行情监控规则（事后告警） | `bid_ask_spread_anomaly_rule` / `price_deviation_from_settlement_rule` / `tick_to_tick_price_jump_rule` |
| `platform-app` | 组装、事件循环、报告 | `main` / `application_configuration_loader` / `component_wiring_and_bootstrap` / `main_event_loop_dispatcher` / `runtime_report_writer` |

源文件用**长英文名 snake_case**（人能看懂在干什么），crate 包名用 **kebab-case 纯英文**，`lib.rs` / `main.rs` 是约定入口，不可改名。

---

## 目录与文件命名约定（重要）

本仓库刻意采用「**ASCII 前缀 + 中文语义后缀**」的目录命名：

```
crates_代码工程/     config_配置/      data_数据/
sdk_ctp库/           logs_运行日志/    reports_运行报告/
storage_持久化存储/  scripts_脚本/     docs_文档/    evidence_测试证据/
```

**为什么这么命名**：本项目面向国内 CTP 期权实盘自用，中文后缀让目录在一年后、十年后仍能被一眼读懂；ASCII 前缀保证 `ls` 排序同类聚簇，且 `grep` / `find` 可敲。

### 安全边界（勿越）

划分红线不是「目录名 vs 文件名」，而是「**路径 vs crate 标识符**」。Cargo / rustc 全程用 `OsString` 处理路径，路径从不进符号表、不进 `.rlib` 命名空间，所以**含中文的路径本身是安全的**；而 crate 名会变成 Rust 标识符、`.rlib` 文件名、macro 导出路径，必须是 ASCII。

| 位置 | 能否含中文 | 说明 |
|---|---|---|
| 目录名 / 文件路径 | ✅ 可以 | 只是 `OsString`，Linux / macOS / Windows-MSVC 均正常，长期维护无碍 |
| `Cargo.toml` 的 `name` | ❌ **禁止** | crate 名是 Rust 标识符 + `.rlib` 文件名 + macro 路径；crates.io 亦只收 ASCII。Cargo **永久不支持**非 ASCII crate 名（`cargo init 中文` 会主动告警），**无稳定化计划** |
| `[dependencies]` 键 / `use crate::` 路径 | ❌ 禁止 | 同上，等同标识符 |
| `.rs` 文件名 | ⚠️ 建议 ASCII | 不致命（Linux / MSVC 可编），但部分旧版 rust-analyzer 索引不稳 |

> **一句话记住**：路径里随便用中文，`name =` 后面永远只能是 ASCII。

### 约定与自检

- 所有 crate 一律 **ASCII kebab-case** 命名（如 `ctp-option-trading-executor`），中文只出现在**目录层与文档层**。
- 新增 crate 时不要顺手 `cargo init 风控引擎`；Cargo 会先跳 warning，看到即改。
- CI / pre-commit 可跑下面这条体检命令自动把关：

```bash
# crate 名必须全 ASCII（非 ASCII 永久不支持）；命中即失败
grep -rn '^name[[:space:]]*=' --include=Cargo.toml . \
  | grep -P '[\x{4e00}-\x{9fff}]' \
  && { echo "❌ 存在非 ASCII crate 名，必须改为 ASCII"; exit 1; } \
  || echo "✅ crate 名全 ASCII"
```

### 已知不兼容环境

- **Windows + `x86_64-pc-windows-gnu`**：gcc / binutils 按 ANSI 代码页解析路径，含中文的路径会报 `linker: Invalid argument`。两选一解决：
  - 换 MSVC 工具链：`rustup default stable-x86_64-pc-windows-msvc`（LLVM 的 lld 无此问题）；
  - 或挂一条纯 ASCII 通路（对 cargo / rustc 完全透明）：

    ```cmd
    mklink /J C:\DevWork\ctp-opt "D:\你的\中文路径\ctp-option-...-platform_CTP期权多前置行情池与风控交易一体化平台"
    ```

    之后一律从 `C:\DevWork\ctp-opt` 编译，顺带绕开「Windows 用户名含中文」那个雷。
- **Docker 构建**：Dockerfile 内显式 `ENV LANG=C.UTF-8 LC_ALL=C.UTF-8`，否则 `COPY` 与 `cargo build` 在非 UTF-8 locale 下可能失败。Linux 原生与 GitHub Actions runner（默认 `C.UTF-8`）无需额外处理。
- **Git 显示中文路径**：`git config --global core.quotepath false`，否则 `git status` 会把中文显示成 `"crates_\344\273\243\347\240\201..."`（文件内容无碍，纯显示层；每台机器各设一次，不入仓库）。

### 为什么不上传 CTP 二进制

`sdk_ctp库/` 与 `data_数据/` 仅保留 `.gitkeep` 占位。CTP API 动态库（`thostmduserapi_se.so` / `thosttraderapi_se.so`）自 6.3.x 起带版权授权条款，不入库，请自行到上期技术官网下载对应版本放入该目录；行情落盘数据同理，走 Release 附件或对象存储，不进 Git 历史。

---

## 技术选型

| 项 | 选择 | 理由 |
|---|---|---|
| CTP 绑定 | **`ctp2rs` v1alpha1 + `ctp_v6_7_7`** | 运行时 dynlib 加载，不污染 `LD_LIBRARY_PATH`；多版本 feature；已在你仿真环境验证 6.7.7 符号对齐 |
| 异步 / 通道 | `crossbeam-channel` + 线程 | 行情回调在 CTP 内部线程，转换后发 channel，主线程汇聚，**不在回调里做 IO** |
| 字符串编码 | `encoding_rs::GB18030` | CTP 所有 `char[]` 是 GB18030，直接用 UTF-8 会乱码 |
| 配置 | `toml` + `serde` | 沿用 `option_monitor` 的配置模式 |
| 错误 | `thiserror` + `anyhow` | 库用 `thiserror`，主程序用 `anyhow` |
| 存储（第一版） | jsonl / parquet | 先不引入时序库，降低运维成本 |
| 存储（后加） | TDengine 或 DolphinDB | 需要曲面/Greeks 流计算选 DolphinDB，只需 tick 持久化选 TDengine |

**明确不用**：`ctp-rs 0.4`（macOS 绑定，Linux 靠手动放 `.so` + `build.rs` 静态链接，换版本要重编）、`rtpx`（RtnTrade 数据错位，已在你仿真验证弃用）。

---

## 路线图

| 阶段 | 内容 | 验收标准 |
|---|---|---|
| **P0** ✅ | 目录 + workspace + 六 crate 空骨架 | `cargo check --workspace` 通过 |
| **P1** | `common-model` + `monitor-rules` 从 option_monitor 平移 + 单测 | 单测通过，行为与 `run_20261017_070057.md` 一致 |
| **P2** | `market-data-connector` 单 member 跑通 ctp2rs 行情登录 + 订阅 | 收到 tick，`contracts.txt` 前 100 合约有数据 |
| **P3** | `multi_front_connection_pool` 多 member 分片 + 主备切换 | `ss -K` 杀一个 member，该片合约 5 秒内迁移到备用 IP |
| **P4** | `trading-executor` 从 rust_ctp_test 抽 `open_session` + insert/cancel | 仿真跑通 2.2 开平撤 |
| **P5** | `risk-control-engine` 三项硬拦截挂在 insert 前 | 超限订单本地被拒，日志 `[RISK REJECT]`，不碰柜台 |
| **P6** | `platform-app` 组装 + 报告定时快照 | 连续跑 2 小时无退出，报告每 5 分钟一份 |
| **P7** | 实盘切换 checklist 逐项过 | 见 `docs_文档/07_Live_Cutover_实盘切换清单_*.md` |

---

## 快速开始

### 环境要求

- Rust 1.75+（`rustup default stable-x86_64-pc-windows-msvc`，Windows 下避免 GNU 工具链踩中文路径）
- Linux（Ubuntu 22.04 已验证）
- CTP 动态库：`libthostmduserapi_se.so`、`libthosttraderapi_se.so`（6.7.7）
- 代理：`cargo` 走 `socks5://127.0.0.1:10808`（见 `.cargo/config.toml`；`no_proxy` 里**不要**含 `127.0.0.1`，否则代理死锁）

### 当前（P0）能跑的

```bash
# 1. 编译检查（空骨架应通过）
cargo check --workspace

# 2. 查看目录
tree -L 3 -I target
```

### 初始化 CTP 库（P2 起需要）

```bash
bash scripts_脚本/init_ctp_sdk_初始化CTP动态库_20261110.sh

# 校验版本，应与 Cargo.toml 的 ctp_v6_7_7 feature 一致
strings sdk_ctp库/lib_动态库/libthosttraderapi_se.so | grep -E 'v?6\.[0-9]+\.[0-9]+'
# 应输出 v6.7.7_xxx
```

### 配置

```bash
cp config_配置/config.example.toml config_配置/config.toml
# 编辑 config.toml，填入真实 BrokerID / AppID / AuthCode / 前置地址
```

`config.toml` **不入 git**，`.gitignore` 已排除。

---

## 配置要点

```toml
[market_data]
# 多前置按站点分组，每组一个 member
[[market_data.front_groups]]
name = "shanghai_telecom_site1"
fronts = ["tcp://101.231.185.21:61213"]
exchanges = ["SSE", "SZSE"]     # 该 member 负责的交易所

[trading]
primary_front = "tcp://101.231.185.21:61205"
backup_fronts = ["tcp://101.231.185.57:61205", "tcp://101.231.185.57:61205"]
settlement_confirm_on_login = true   # 每天首次登录必须确认，否则报单 42

[risk]
max_notional_per_order = 500000.0
max_position_per_instrument = 50
max_cancel_rate_per_minute = 20
self_trade_prevention = true
```

完整字段见 `config_配置/config.example.toml` 注释。

---

## 已知约束与风险

### CTP 业务坑（已踩过，已记录在 docs 07/08）

- **报单成功不看 `OnRspOrderInsert`**：成功走 `OnRtnOrder` 且 `sys_id` 非空；`OnRspOrderInsert` 只在拒绝时回调。
- **上期所撤单必须本地会话定位**：`FrontID + SessionID + OrderRef`，`OrderSysID` 留空；填两组反而报 25。
- **结算结果未确认**：每交易日首次登录必须 `ReqSettlementInfoConfirm`，否则报单 42。
- **涨跌停以柜台为准**：仿真涨跌停与实盘差异巨大，必须从 `OnRtnDepthMarketData` 的 `UpperLimitPrice/LowerLimitPrice` 取。
- **GB18030 编码**：所有 CTP `char[]` 读取用 `encoding_rs::GB18030`，否则错误消息乱码。
- **`no_proxy` 含 `127.0.0.1` 会让 cargo 代理死锁**：代理地址本身是 `127.0.0.1` 时，`no_proxy` 把它排除，cargo 连不上。

### 实盘切换前必须确认（P7）

1. BrokerID / AppID / AuthCode 三件套与实际报备一致。
2. 看穿式终端信息：实盘 `ReqAuthenticate` 会校验 Mac/IP/硬盘序列号，仿真宽松。
3. 流控：实盘每秒报单/查询有硬限制，不能循环压测。
4. 跨日重连后重新结算确认。
5. 资金/持仓：先查 `ReqQryTradingAccount` / `ReqQryInvestorPosition`，风控阈值按实际资金设。

---

## 文档索引

| 文档 | 内容 |
|---|---|
| `docs_文档/01_Architecture_架构总览_20261110.md` | 四层架构、数据流、crate 边界 |
| `docs_文档/02_Market_Data_Pool_行情池设计_20261110.md` | 连接池状态机、分片策略、主备切换 |
| `docs_文档/03_Trading_Executor_交易执行层_20261110.md` | 会话生命周期、报单/撤单、重连 |
| `docs_文档/04_Risk_Control_风控规则_20261110.md` | 四项硬拦截的阈值与触发条件 |
| `docs_文档/05_Error_Codes_错误码_20261110.md` | 平台错误码 `E_XXXX` 与 CTP ErrorID 对照 |
| `docs_文档/06_Workflow_工作流_20261110.md` | 脚本命名、提交规范、质量门禁 |
| `docs_文档/07_Live_Cutover_实盘切换清单_20261110.md` | 实盘前逐项检查 |

---

## 贡献与提交规范

提交前缀：`feat` / `fix` / `refactor` / `docs` / `chore` / `perf`。

正文必须包含：

```
为什么改：
- ...
改了什么：
- ...
修正了之前的：
- ...
```

质量门禁（pre-commit）：`cargo fmt --check` / `cargo clippy --all-targets -- -D warnings` / `cargo test`。

---

## 许可与免责

[Apache License 2.0](./LICENSE)。

**本项目涉及真实资金交易接口。** 仿真环境跑通不等于实盘可用。实盘上线前必须完成 `docs_文档/07_Live_Cutover_实盘切换清单_*.md` 全部项，并在小流量单品种验证后再逐步放大。作者不对任何交易损失负责。

---

<p align="center">
  <sub>Built with Rust · CTP 6.7.7 · 行情要稳，交易要通，风控要拦在报单之前</sub>
</p>
