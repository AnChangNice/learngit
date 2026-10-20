# OPP 实现审查问题合集（pro + flash 合并报告）

> 审查对象：分支 `pr-119118` 的最近两条 commit
> - `38b53d321a1` "bluetooth: classic: add OPP profile"（核心 + 公共 API，+1563 行）
> - `b10cbd80ffe` "bluetooth: classic: shell: add OPP shell commands"（shell，+1047 行）
>
> 来源：两个并行审查任务（deepseek-v4-pro 与 deepseek-flash），各自用 6 维度 finder + 对抗验证的工作流跑完。本文档合并两边的最终报告与工作流结果，按独立问题去重。
>
> 口径：
> - 严重度取双方判定中较高者，并标注两边原判；两边判定相反的问题单独列为 B 节。
> - 所有结论均来自静态审查 + `qemu_x86` 编译验证，**没有在两台真实设备/bsim 上跑过 OPP 互操作**。
> - pro 工作流终态 36 成立（24 CONFIRMED + 12 PARTIAL）/ 3 推翻；flash 终态 24 confirmed / 1 disputed / 2 rejected（27 条含重复，约 21 个独立问题）。

## 统计总览

| 分类 | 数量 | 占比（并集 46 个独立问题） |
|---|---|---|
| A. 双方一致确认 | 12 | 26.1% |
| B. 双方都提到、结论相反 | 2 | 4.3% |
| C. pro 独有 | 25（含 3 条被自己推翻） | 54.3% |
| D. flash 独有 | 7 | 15.2% |

| 指标 | pro | flash |
|---|---|---|
| finder 原始发现 | 42 | 27 |
| 去重后 | 39 | 24（2 条被验证上限截断丢弃） |
| 验证 | 39 条全部，每发现 1 个验证器 | 22 条主轮（每发现 2 个验证器投票）+ 5 条 gap 轮 |
| 终态 | 36 成立 / 3 推翻 | 24 confirmed / 1 disputed / 2 rejected |
| agent / 耗时 | 45 / ~22 min | 61 / ~42 min（恢复后） |

## 修复优先级建议

- **P0（必修，双方共证或 major 级）**：A1、A2、A3、A11、D1、C1-1、C1-2
- **P1（应修）**：A4、A5、A6、A7（含 A12 的回滚）、A9（补测试）、A10（@since / release notes）
- **P2（文档 / 加固）**：D2、D3、D5、D6、D7、C3 全部、C4 大部分
- **不修 / 降级**：B1（防御性加固即可）、B2（设计层面建议，非本 PR 引入）

---

## A. 双方一致确认（12）

### A1. 客户端 DISCONNECT 被拒时状态错置为 DISCONNECTED `(major, 双方)`

`subsys/bluetooth/host/classic/opp.c:97`

`opp_client_obex_disconnect_cb()` 无视 `rsp_code` 无条件置 DISCONNECTED；OBEX 层在拒绝 DISCONNECT 时保持会话（`obex.c:1231-1236` 仅 SUCCESS 置断开），树内 map.c/pbap.c 均恢复 CONNECTED，OPP 自己的 server 侧（`opp.c:601-605`）也注释了拒绝断开应保持连接。后果：对端回非 SUCCESS（如 0xC3/0xCF）后 OBEX 会话仍活着、OPP 状态已断开 → push/pull/abort/disconnect 全 -ENOTCONN，重连在 OBEX 层 -EINPROGRESS 失败，**只能拆 transport 重连**。

修复：`atomic_set(&client->_state, rsp_code == SUCCESS ? DISCONNECTED : CONNECTED)`（对齐 obex.c / pbap.c:373-378）。

### A2. transport 重复注册把活动 RFCOMM accept 置 NULL → 入站连接空指针崩溃 `(major, 双方)`

`subsys/bluetooth/host/classic/opp.c:514`

`bt_opp_server_transport_register()` 每次调用先无条件执行 `server->server.rfcomm.accept = NULL`。成功路径上这行是死代码（`goep.c:246` 会覆盖）；但重复注册时 GOEP 在 `goep.c:241-244` 先返回 -EEXIST、**不恢复** accept，而 RFCOMM 的 server 仍挂在 `servers` 链表（`rfcomm.c:202`），下一条入站连接在 `rfcomm.c:624` 无 NULL 检查地调用 `server->accept(...)` → **崩溃**。被毁掉的值还是 goep 自己的 `goep_rfcomm_accept`（goep.h 明确要求应用不得设置该字段）。pro 工作流原判 minor、报告升级为 major。

修复：删掉 `opp.c:514` 这行；重复注册返回 -EALREADY 且不动原注册。

### A3. server 实例注册一次后不可复用 / `cb` 被清 / 活连接被重新初始化 `(major, 双方)`

`subsys/bluetooth/host/classic/opp.c:544`（pro M2 + `opp.c:493` minor）；flash：`opp.c:534`（major）+ `opp.h:737`（disputed）

`bt_obex_server_register()` 把 `_node` 挂进 `_goep.obex._servers`（`obex.c:1755-1758` 重复注册返回 -EALREADY），而 transport 断开时 `obex.c:1699-1705` 只重置状态、**不摘节点**，OPP 也没有 unregister。两个具体后果：

1. transport 已接入但 OBEX CONNECT 未到时 `_state` 仍是 DISCONNECTED，第二次 `bt_opp_server_register()` 通过检查后被 OBEX 层拒 -EALREADY，`opp.c:544-547` 顺手 `cb = NULL` → 下一次 CONNECT/transport 事件在 `opp.c:346/360/385` 解引用 `server->cb` → **崩溃（远端可触发 DoS）**。
2. 若第一条 OBEX 连接已建立，第二次调用 -EINPROGRESS 不清 cb；accept 回调忽略该错误返回 0 时（其唯一校验是 `cb != NULL`，`opp.c:493-496`），`BT_GOEP_INIT_V1()`/`goep_rfcomm_init()` 重新初始化活实例 → 同一 DLC 节点被 prepend 进第二个 session 的链表（`rfcomm.c:607`），两个会话互串。

shell 靠每次 accept 前 memset 规避（`shell/opp.c:779`），memset 本身还会破坏链表语义。pro 侧 `opp.c:493`（`cb != NULL` 不能证明实例可用，-EINPROGRESS 提前返回时 cb 保留旧值）为同族问题的另一面。

修复：register 时同时检查 `_transport_state`；-EALREADY 时**不清 cb**；accept 里要求实例空闲（OBEX state DISCONNECTED）才允许 `BT_GOEP_INIT_V1()`；提供 `bt_opp_server_unregister()`（学 bip.c:1213-1239）或在 register 内部重置内嵌 OBEX server。

### A4. shell pull 收到 CONTINUE 后不续传，客户端卡在半途 `(major, flash / minor, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:168`

`opp_cli_pull()` 接受 CONTINUE、打印第一块 Body 就返回，从不再次 `bt_opp_client_pull()`——违反 `opp.h:432-436` 文档契约。OBEX 层 CONTINUE 故意保持操作活跃（`obex.c:1424-1426`），之后 push/disconnect 全 -EBUSY（`obex.c:2230-2233, 2057-2059`），只能拆链或手滑重跑 pull 碰巧恢复。

修复：CONTINUE 时用 `bt_opp_client_create_pdu(client, NULL)` 分配空 buffer 再 pull，循环到非 CONTINUE；`opp exchange` 的 pull 阶段同样处理。

### A5. malformed 2 字节 Name 被当作空 Name `(minor, 双方)`

`subsys/bluetooth/host/classic/opp.c:435`

`name_len <= 2` 即视为空 Name（注释称"2 字节即空"），但只有 `{0x00,0x00}` 才是空 UTF-16BE 串。`{0x00,0x41}`（"A"）或 1 字节值会穿透检查，对带名对象的 GET 返回默认名片，违反 `opp.h:583-587` 承诺的"非空 Name 回 FORBIDDEN"（接收路径无 UTF-16 校验，`obex.c:3792` 返回原始值）。

修复：只放行 absent / len==0 / 恰好 `{0x00,0x00}`；奇数长度与其他 2 字节值按非空请求处理（FORBIDDEN/BAD_REQUEST）。

### A6. exchange pending 标志竞态：漏触发或污染下一次 push `(minor, pro 报告 / minor, flash)`

`subsys/bluetooth/host/classic/shell/opp.c:521`、`opp.c:114`

两个方向：① flag 在 `bt_opp_client_push()` **返回后**才置位（521 行），对端抢先处理完 PUT 响应（RX 线程）则 exchange 的 pull 阶段被静默跳过；② flag 只在 SUCCESS 分支清除（114-120 行），被拒的 exchange（FORBIDDEN 等）把 flag 留给下一次普通 push → **计划外的 GET**。pro 工作流原判 major，报告里降为 minor。

修复：push 前置位（push 出错路径回滚）；所有终态响应（非 CONTINUE）都清 flag。

### A7. shell 硬编码 RFCOMM channel 5（SPP 保留通道）`(major, flash / minor, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:552`

`OPP_SHELL_RFCOMM_CHANNEL = 5U` 是 `BT_RFCOMM_CHAN_SPP`（`rfcomm.h:58-66` 明确 1-5 保留给 HFP/HSP/SPP）。同镜像里 SPP 占用 5 时 `bt_rfcomm_server_register()` 返回 -EADDRINUSE（`rfcomm.c:194-197`）；且 SDP 记录在 transport 之前注册（`opp.c:802`）、失败不回滚（见 A12），远端会发现在 channel 5 上广告 OPP 却由别的服务监听。

修复：channel 0 动态分配（`BT_RFCOMM_CHAN_DYNAMIC_START`），SDP 记录引用注册后的运行时 channel（学 pbap.c:819, 857-859, 1188-1194）；或给 OPP 加专用保留通道。

### A8. 注释引用不存在的 `opp_push_server` sample `(minor, flash / nit, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:549-551`

"Channel 5 is used by the opp_push_server sample"——全树（含 git 全历史）无此 sample，`grep samples/ BT_OPP` 为空。违反"注释描述树中现状"。

修复：删掉该引用，或补一个真正的 sample（contributor_expectations 也要求新 API 配示例）。

### A9. 851 行公共 API + 640 行状态机零功能测试 `(major, flash / minor, pro)`

`tests/bluetooth/shell/prj_br.conf:52-53`

OPP 只被编译、从不被执行：`bluetooth.shell.shell_br` 及两个变体都是 `harness: keyboard`，twister 只编译不运行（keyboard harness 无实现）；无 `tests/bluetooth/classic/opp`、无 bsim 覆盖、无 sample。任何 OPP 线格式/状态机回归都能通过全部 CI。

修复：`tests/bluetooth/classic/opp` + pytest/usb_hci bumble 对端（学 sdp_s/rfcomm_s）；至少覆盖 CONNECT（无 Target）、单包 PUT、默认对象 GET、多包 CONTINUE、ABORT；顺带让多包路径可达（见 A4）。

### A10. `@since` 版本号与 release notes `(major, pro / minor, flash)`

`include/zephyr/bluetooth/classic/opp.h:17`

- pro（基于分支状态，VERSION 4.5.0-rc1）：`@since 4.6` 错误，本周期合入应为 4.5；同周期 ascs.h/hci_pkt.h 均用 `@since 4.5`，文档工具会把 4.6 渲染成不存在的 v4.6.0 tag。
- flash（基于 main，VERSION 4.5.0-rc2）：main 上已有 `doc/releases/release-notes-4.6.rst`，其 New APIs 的 keep-sorted 块为空，两条 commit 都没动 `doc/`，违反 AGENTS.md 要求。

两者合起来看：**取决于 rebase 后落进哪个周期**——进 4.5 则改 `@since 4.5` + 加 release-notes-4.5 条目；进 4.6 则保留 `@since 4.6` + 往 release-notes-4.6.rst 的 keep-sorted 块补 `bt_opp` 与 `CONFIG_BT_OPP_SERVER/CLIENT` 条目。

### A11. `create_pdu()` 池耗尽时 `K_FOREVER` 阻塞而非返回 NULL `(major, pro / minor, flash)`

`subsys/bluetooth/host/classic/opp.c:121`（死锁场景）；`opp.h:330-331, 691-692`（契约面）

文档承诺"no buffer is available"返回 NULL，实际经 `bt_rfcomm_create_pdu → bt_conn_create_pdu` 用 `K_FOREVER`（`conn_internal.h:602-610`），只有系统 workqueue 降级 K_NO_WAIT（`conn.c:1604-1608`）。pro 的深挖结论：`opp_cli_push` 在 BT RX 线程同步执行，exchange 自动 pull 时在 RX 线程做 K_FOREVER 分配；池仅 `2*CONFIG_BT_MAX_CONN`，一次 exchange 可瞬时占用 3 个（在途 push PDU 等 HCI tx-complete + server 侧 DLC tx_queue 卡住的响应 PDU）→ 第三次分配永久阻塞 → 释放前两个 buffer 所需的 tx-complete 事件与 RFCOMM credit 帧永远无法处理 → **BT RX 路径死锁**（goep.c:387-391 自注释过此坑）。flash 判定为文档级 minor。

修复：改文档 + 加 `k_timeout_t` 参数（学 `bt_a2dp_stream_create_pdu()`），或自动 pull 丢 workqueue / 改 K_NO_WAIT，或按瞬时在途量放大池。

### A12. transport 注册失败后 SDP 记录残留 `(major, flash / 折入 A7, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:802-812`

`cmd_opp_server_register()` 先注册 SDP（802）、后注册 transport（808），失败直接返回**不回滚** `bt_sdp_unregister_service()`；记录留在静态 `sdp_db`，远端持续发现 channel 5 上的 OPP 服务却连进别的监听器（典型触发：先 `spp register_with_channel 5` 再 `opp server_register`）。同树 SPP shell 有正确回滚范式（`shell/spp.c:363-367`）。

修复：失败时 `bt_sdp_unregister_service()` 并记录回滚失败；不要无条件把 -EEXIST 当成功（可能是上次失败留下的记录）。

---

## B. 双方都提到、结论相反（2）

### B1. `opp.c:782` accept 里 `in_use` 泄漏 `(pro: minor confirmed / flash: 2 验证器 rejected)`

`subsys/bluetooth/host/classic/shell/opp.c:775-782`

pro 单验证器确认"register 失败时 `in_use` 永真、单例 wedged"。flash 的双验证器判定**不可达**：memset 之后 `bt_opp_server_register()` 在当前树没有失败路径（回调齐全、状态为 0、uuid 允许 NULL），且 transport_disconnected 会清 `in_use`。

**采纳 flash 的判定**：降级为防御性加固（失败路径回滚 `in_use`），不是 live bug。

### B2. `opp.h:719` 无 unregister/teardown API `(pro: 作为 A3 一部分, major 语境 / flash: rejected)`

`include/zephyr/bluetooth/classic/opp.h:719`

flash 判定：GOEP 层 `goep_rfcomm_server` 列表本身就没有摘除函数（`goep.c:241-254`），PBAP/MAP/BIP 同样，属既有架构限制、非本 PR 引入；但 **server 实例 re-register 失败（A3）仍然成立**。pro 把它作为 A3 的根因之一。

**采纳 flash 的判定**：不在本 PR 引入 teardown API；A3 的崩溃按 A3 修复，teardown 作为 API 设计建议另行讨论。

---

## C. pro 独有（25 = 22 确认 + 3 自推翻）

### C1. 并发 / 时序（2 major，第 3 个已在 A11）

#### C1-1. RX 线程 memset `g_shell_server_inst` 与 shell 线程 rsp 命令竞态 `(major, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:779`

accept 回调在 RX 线程 memset 单例（`in_use` 由另一线程的 disconnected 回调清除），而 5 个 `cmd_opp_server_*_rsp` 只做 check-then-use。交错：shell 线程通过 `in_use` 检查（847 行）→ 对端 A 断开、B 连入并 memset+重注册 → shell 线程继续执行 `bt_obex_connect_rsp`：memset 落在 `obex.c:1928` 的 NULL 检查之后、`obex.c:1938/2022` 解引用之前 → **NULL 解引用崩溃**；memset 抢先完成 → 给 A 的响应发到 B 的连接上。

修复：accept 里新分配实例（或池）+ generation/epoch 计数，或把 accept 与 rsp 串行化到同一线程。

#### C1-2. `opp_cli_transport_active` 丢失更新 → 客户端永久卡死 `(major, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:283`

flag 在 `bt_opp_client_transport_connect()` **返回后**才置 true（283 行）；RFCOMM 连接异步，对端立即断开时 disconnected 回调（workqueue 线程）先写 false（91 行），shell 线程随后执行 283 行又写回 true——没有 live transport 却 flag=true。此后所有 `opp transport_connect` -EALREADY、`transport_disconnect` 因 CAS 需要 CONNECTED 而 -ENOTCONN，**没有任何命令能清 flag**。

修复：调用前置位（真正的守卫是 `opp.c:164-167` 自己的 CAS）、改 atomic，或删掉 flag 直接从 `_transport_state` 推导。

### C2. 状态机 / 生命周期（3 minor）

#### C2-1. server 从不进入 DISCONNECTING 中间态 `(minor, pro)`

`subsys/bluetooth/host/classic/opp.c:390`

收到 DISCONNECT 请求后直接处理，不进 `opp_internal.h:24-25` 定义好的 DISCONNECTING 态（map.c 有先例），拒绝断开时缺 CONNECTED 回退分支。影响限于内部一致性，nit 级。

#### C2-2. server 五个响应 API 无原子"认领"，可双发响应 `(minor, pro)`

`subsys/bluetooth/host/classic/opp.c:570`

obex 层用分离的 atomic_get 检查 state/opcode、send 成功后才清 opcode（`obex.c:1947-1955, 2030`）——RX 线程与 app 线程并发响应同一请求会双发（shell 的自动 FORBIDDEN 回复与用户命令就可能撞车）。对比客户端请求路径是 `atomic_cas` 认领的（`obex.c:2228`）。

#### C2-3. 单 server 实例可被 wedged，无 `server_close` 命令 `(minor, pro)`

`subsys/bluetooth/host/classic/shell/opp.c:780`

对端连上后既不发请求也不断开，`in_use` 永真、后续连接全被拒（-ENOMEM），shell 无任何命令能关闭该实例。

### C3. API 契约（5）

#### C3-1. 文档承诺"出错时调用者保留 buffer"，实际 buffer 已被修改 `(minor, pro)`

`include/zephyr/bluetooth/classic/opp.h:343`

所有发送 API 承诺出错时调用者保留 buffer；但 obex.c 已把 3 字节请求头 `net_buf_push` 进调用者的 buffer 且失败时**不还原**（`obex.c:2277-2280, 2292-2300`）。按文档重试同一 buffer 会双重压头、发出畸形包。要么修 obex.c 保存/恢复 `net_buf_simple` 状态，要么文档写明失败后 buffer 已被修改。

#### C3-2. MOPL 约束未文档化且静默钳制 `(minor, pro)`

`include/zephyr/bluetooth/classic/opp.h:347`

<255 返回 -EINVAL、超过 transport MTU 被静默钳制，两者都没写；shell 传 0xffff 实际发的不是 0xffff。

#### C3-3. `-EALREADY`/`-ENOTCONN` 的 @retval 描述不含 in-progress 状态 `(nit, pro)`

`include/zephyr/bluetooth/classic/opp.h:292`

CONNECTING/DISCONNECTING 时同样返回这些码，文档未提。（flash 被验证上限截断丢弃的 2 条里正好有这条——独立印证。）

#### C3-4. `opp.h` 不自包含 `(nit, pro)`

`include/zephyr/bluetooth/classic/opp.h:23`

直接用 `BT_OBEX_RSP_CODE_*`/`struct net_buf`/`BT_UUID_DECLARE_16`，只 include kernel.h 和 goep.h，靠传递包含。

#### C3-5. `bt_opp_rsp_code` 只列 13/39 个 OBEX 响应码 `(nit, pro)`

`include/zephyr/bluetooth/classic/opp.h:90`

所有回调文档都指向它，但对端可合法发 UNAUTH/NOT_ALLOW/DB_FULL 等，应用无从命名。map.h 的做法是回调参数直接用 `bt_obex_rsp_code`。

### C4. 规范符合性 / shell 健壮性（12）

| # | 位置 | 级别 | 问题 |
|---|---|---|---|
| C4-1 | `opp.c:288` | minor | push 强制 End-of-Body，却不校验自己 Doxygen 声称"必须包含"的 Name/Length 头（OPP 4.2.2/5.5）；无名字无长度的 PUT 可端到端成功 |
| C4-2 | `opp.c:953` | minor | pull 成功响应只有 Length + End-of-Body，缺 OPP Table 5.3 对 Push Server 标为 Mandatory 的 Type 头（text/x-vcard） |
| C4-3 | `opp.c:701` | nit | SDP 广告 Supported Formats List，server 却从不校验收到的 Type，可接受广告之外的类型 |
| C4-4 | `opp.c:66` | minor | SDP 发现多条记录时"最后一条赢"且无选择手段（map shell 是打印每条让用户传 channel）；`uint16→uint8` 截断无检查，channel 261 静默变 5 |
| C4-5 | `opp.c:326` | minor | 错误态只报原始 errno（"err -128"）；OBEX 无 connect 响应超时，对端沉默时只能 transport_disconnect |
| C4-6 | `opp.c:310` | minor | `strtoul` 无 endptr/范围校验：`connect 65536` 静默回绕、`connect abc` 变 mopl=0 的迷惑报错 |
| C4-7 | `opp.c:239` | minor | `opp_rfcomm_channel` 在 SDP 回调（RX 线程）与 shell 线程间无同步，连发两次 discover 会用过期通道拨号或误报"no OPP service" |
| C4-8 | `opp.c:852` | nit | 三个 server rsp 命令的 keyword→响应码解析链重复三份且已漂移（push 有 toobig/unsupported、pull 有 notfound、connect 都没有）；vCard 载荷重复两份 |
| C4-9 | `opp_internal.h:1` | minor | 无 include guard（MISRA Dir 4.10 / PRE06-C），目前只有 opp.c 一个包含者所以能编译 |
| C4-10 | `opp.c:218` | nit | `ascii_to_utf16be` 超长静默截断、非 ASCII 字节拆出 mojibake UTF-16 |
| C4-11 | `opp.c:185` | nit | 对端可控 body 用 `%.*s` 直打终端：UTF-16 内容遇 NUL 只打首字符，二进制可注入控制序列 |
| C4-12 | `opp.c:115` | nit | `SUCCESS \|\| OK` 比较是死代码（两者都是 0xA0） |

### C5. pro 检查后自己推翻的疑点（3，不再作为问题）

- `opp.c:274` — `memset(&opp_client)` 在活跃 RX 回调下清零：被 `opp_cli_transport_active` 守卫 + transport 断开时序挡住，**不成立**。
- `opp.c:78` — 对端发 OBEX 1.0 的 CONNECT 成功码 0x20 会被当失败：OBEX 层做了归一化，**不成立**。
- `opp.c:136` — pull 请求带空 Name 头是对 OPP Table 5.3 的误引：注释的 spec 引用成立，**不成立**。

---

## D. flash 独有（7）

### D1. server 分发 pull 回调时丢掉 OBEX Final 位 `(major + minor, flash)`

`subsys/bluetooth/host/classic/opp.c:447`；`include/zephyr/bluetooth/classic/opp.h:601`

`opp_server_obex_get_cb()` 收到 `final` 参数（413 行）只打了 LOG，`bt_opp_server_cb.pull()` 签名只有 `(server, buf)`。OBEX 明确允许 GET 请求阶段跨包（`obex.h:790-793`），非 final GET 时 `obex_server_get_common()` 存 opcode=GET 并回调；此时应用按文档回 SUCCESS 会被 `bt_obex_get_rsp()` 以 -EINVAL 拒绝（"Success cannot be responded without final"，`obex.c:2529-2532`）→ 对端超时；回 CONTINUE 则最后一包也收 CONTINUE、传输永不完成。树内 pbap.c:1003、bip.c:1052、OPP 自己的 push（opp.h:576）都向应用暴露 final。

修复：`bt_opp_server_cb.pull()` 加 `bool final` 参数（对齐 push），文档写明非 final 必须回 CONTINUE；或在 OPP 层拒绝非 final GET。

### D2. server 的 register 文档称"初始化实例"，实际要求调用方先零初始化 `(major, flash)`

`include/zephyr/bluetooth/classic/opp.h:724`

`bt_opp_server_register()` 不零化对象：依赖 `_state == 0` 和内嵌链表为空，未 memset 的 `k_malloc` 实例会让 `sys_slist_find/append` 解引用垃圾。goep.h:248/307 对同一要求有 `@warning`，opp.h 没有；断开后的实例也无法再次注册（与 A3 同根）。

修复：struct 与 register 函数加 `@warning` + `-EALREADY` 文档；配合 A3 的复用方案。

### D3. client 零初始化契约缺失 `(major, flash)`

`include/zephyr/bluetooth/classic/opp.h:295`

`opp.c:164-167` 的 CAS 依赖 `_transport_state == 0`；栈上/`k_malloc` 未清零实例会得到误导性的 -EALREADY（文档称"Transport has been connected"），或 `bt_obex_connect()` 遍历未初始化的 `_clients` 链表。shell 靠 connect 前 memset 规避（`shell/opp.c:266-274`）。

修复：`struct bt_opp_client` 与 `bt_opp_client_transport_connect()` 加 `@warning`，-EALREADY 描述覆盖 CONNECTING/CONNECTED/DISCONNECTING。

### D4. SDP 广告 OPP 1.0（0x0100）而非 1.1 `(minor, flash)`

`subsys/bluetooth/host/classic/shell/opp.c:619`

`BT_SDP_ARRAY_16(0x0100)` 注释写 `/* v1.1 */`——SDP 版本编码为 major<<8|minor，0x0100 是 1.0；树内对照：MAP 1.4 = 0x0104、PBAP 1.2 = 0x0102、BIP 1.2 = 0x0102、HID 1.1 = 0x0101。依赖版本号的远端会按 1.0 处理或跳过服务。

修复：`BT_SDP_ARRAY_16(0x0101)`，最好定义共享的 `BT_OPP_VERSION` 常量。

### D5. abort 回调文档承诺与实现不符 `(minor, flash)`

`include/zephyr/bluetooth/classic/opp.h:459`

文档称"server 响应时调用 abort 回调"；实际带 header 的 ABORT 响应被分发给**前一个 opcode** 的 handler（`obex.c:1539-1545`）、非 SUCCESS 直接断 transport（`obex.c:1558-1564`），只有空 SUCCESS 响应才真正调用 `ops->abort`。按文档等待 abort 回调的应用在拒绝场景永远等不到，只观察到 transport_disconnected。

修复：改文档如实描述，或在 `opp_client_obex_abort_cb` 处理错误/带 header 场景兑现承诺。

### D6. Default Get Object 不校验 Type 头 `(minor, flash)`

`subsys/bluetooth/host/classic/opp.c:446`

文档保证 pull 回调的 Type 必为 `BT_OPP_TYPE_VCARD`（opp.h:581-587），实现只查 Name、从不读 Type：`text/x-vcalendar` 甚至无 Type 头的 GET 也按名片拉取处理。`bt_obex_get_header_type()`（obex.c:3821）存在但未使用。

修复：读取并校验 Type（非 vCard 拒绝并文档化响应码），或从文档删除该保证、声明由应用校验。

### D7. bt_opp 无 docs 页面 / toctree 条目 `(minor, flash)`

`include/zephyr/bluetooth/classic/opp.h:16`

40 符号的公共 API 组 + 15 条 shell 命令，两条 commit 都没动 `doc/`：Bluetooth Classic API 页（`doc/services/connectivity/bluetooth/api/index.rst`）止于 bip.rst、Bluetooth Shell 参考页无 opp 条目（MAP/SPP 近期合入时都补了）。API Overview 表会因 `api-overview-table` 自动列出该组，但 curated 页面缺失。

修复：新增 `doc/services/connectivity/bluetooth/api/classic/opp.rst`（`.. doxygengroup:: bt_opp`）与 `doc/services/connectivity/bluetooth/shell/classic/opp.rst`，并加入对应 toctree。

---

## 附录：来源

- pro 任务会话：`C:\Users\Chang\.claude\projects\E--repos-zephyrproject-zephyr\f4f25688-...jsonl` → 续会话 `7ce7e3ac-...jsonl`；工作流结果 `...\7ce7e3ac-...\workflows\wf_9345630a-e31.json`（45 agent，~22 min）
- flash 任务会话：`8441cf8d-...jsonl` → 续会话 `237441b4-...jsonl`；工作流结果 `...\237441b4-...\workflows\wf_8db28239-080.json`（61 agent，~42 min；第一次运行在 5/6 处 checkpoint 失败后手动恢复）
- flash 逐条 journal：`...\237441b4-...\subagents\workflows\wf_8db28239-080\journal.jsonl`
- pro 报告文本：`C:\Users\Chang\AppData\Local\Temp\pro_report.txt`；flash 报告文本：`C:\Users\Chang\AppData\Local\Temp\flash_report.txt`
- 构建验证：`west build -b qemu_x86 tests/bluetooth/shell -- -DCONF_FILE=prj_br.conf` 通过，0 警告（pro 执行）；checkpatch 两条 commit 均 0 错误
