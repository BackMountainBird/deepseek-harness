# Agent Note: Mux 扇出共享化，每帧只序列化一次

Status: implemented

[English](2026-08-19-mux-fanout-serialize-once.md) | 中文

## 问题

一个运行大量并发 agent（智能体）的 `dsh web` 会话（一个用六到八个并行 subagent 调研仓库的工作流）把宿主进程主线程占满一个核心的 100%，RPC 读取（`subagent.list`、`session.list`）则从约 0.5 秒恶化到约 105 秒。在复制的 session 存储上隔离复现：只有一个流式会话且无连接时 CPU 约 20%、读取 0.6 秒；持有十二条 WebSocket 连接把同样负载推到约 76% CPU，一次 `subagent.list` 达 82.7 秒。这个差距是一条饱和悬崖——loop 一旦接近满载，列举中每个被 await 的文件操作都要排在它后面。

扇出核算显示 mux 广播把每消费方的工作在三个方向上成倍放大（全部在实况服务器与隔离复现上测得）：

- `session/event`、`session/created`、`session/disposed` 与 jobs 监听器**每条打开的流**都注册一次，因此每个流式事件都要跑一遍 `viewFor`（包括 presenter 调用与调用配对回扫），并为每条连接铸造一个新 rpcId。
- 随后每个载体各自重新序列化帧：WebSocket 下行对每 socket 每帧调用一次 `JSON.stringify`，SSE 载体对每条 GET 流做同样的事。
- 因此一次对十二个消费方的广播，除 socket 写入外一切都付十二倍。在复现中，六个会话二十分钟向十二条连接推送了 43.1 万帧 / 150 MB。

## 决策

**一次铸造、一次序列化、N 次写入。** `EventsApi` 约定现在产出 `StreamFrame<F> = { request, wire }`：帧的 server-request 视图，加上在推送时计算一次的 JSON 字节。`broadcast()` 构建一个 `StreamFrame` 并把同一个 item 对象推入每个队列；两个载体（WebSocket 下行与 fetch handler 的 SSE 路径）逐字发送 `frame.wire`。协议字节不变——浏览器解析的内容与之前完全相同；移动的只是宿主内部的 seam 类型。

**共享监听器，在首次打开流时注册。** 四个热路径监听器从 `events.mux()` 移出到代理级的 `registerMuxFanout()`，在首次打开时调用。打开调用配对表（`openCalls`）是共享的——配对是会话事件流的属性，不属于任何一条连接；表未命中仍走回扫。注册是惰性的，原因有二，两者都承重：

- **零消费方成本严格为零。** 没有打开流的代理完全不注册 `session/event` 监听器。早期草稿在代理 setup 时急切注册；在覆盖率插桩下，仅每事件守卫就把一个搜索测试推过了 5 秒超时（基线通过；急切版本失败），而且无流的代理本来就没有理由观察事件总线。
- **`ctx.inject` 异步激活。** 通过 `ctx.inject(['jobs'], …)` 注册 jobs 监听器会错过代理创建后同一 tick 内发生的注册表变更（jobs spec 的 start/kill/settle 序列在 inject 子上下文激活前触发，因此只有 settle 推送到达）。惰性注册在首次打开时同步读取 `ctx.get('jobs')`——与每流代码相同的时序——更晚组合的 jobs 注册表会在下一次打开时被拾取。

最后一个消费方离开后，监听器保持注册，但在任何帧铸造、presenter 工作或序列化之前，对 `muxQueues.size === 0` 直接空转返回。

**每连接状态仍是每连接的。** 打开时的基线（已订阅帧、带稳定 rpcId 的待定 approval/question 重放、队列快照、jobs 快照）仍按流铸造和序列化，频率即打开频率。空集 jobs 推送在非自有注册表变更时仍发送 `[]`——这一转换是「缺席」无法表达的。

## 后果

对每个流式事件的每个消费方，剩余成本是对预计算字节的一次 `socket.send`。`session/jobs` 空快照语义、approval/question 重放约定与协议格式不变，由既有套件覆盖；一个新 spec 直接断言共享约定（两个消费方收到同一个 `StreamFrame` 对象——引用相等——其 `wire` 可解析回该帧的 server-request 信封），并断言没有打开的流时事件到不了任何消费方。客户端 fixture 在进程内 tap 上通过 `serverRequestSchema` 解析每帧的 `wire` 来镜像该约定，因此现在每条 fixture 旅程也会演练序列化形式。消费方侧的 `IApiClient` 表层不变：它仍产出从 wire 解析出的 `RpcRequest` 信封，浏览器 bundle 看不到 `StreamFrame` 类型。

## 考虑过的替代方案

- **载体内的每连接 WeakMap 序列化缓存**——本可在不动约定的情况下修复 `JSON.stringify` 重复，但把每连接监听器重复（viewFor + rpcId 铸造，扇出成本的大头）留在原地，并把共享隐藏为载体内部的隐式优化而非约定。
- **急切的代理级监听器**——注册更简单，但给每个无流代理增加每事件成本；因上文测得的覆盖率运行回归而否决。
- **帧批量化／每连接订阅过滤**——收益更大但复杂度更高（批量化要改协议格式；过滤要引入订阅协议）。推迟到共享扇出的效果在真实部署上测得之后再评估。

## 测量附录

隔离复现（复制的 session 存储、同一二进制）：空闲服务器 2% CPU、`subagent.list` 0.54 秒；一个流式会话、无连接 30% CPU、0.6 秒；六个会话 + 十二条连接 76% CPU、`subagent.list` 82.7 秒（带工作流波次的实况服务器：100% CPU、105 秒）。扇出共享移除了每连接序列化与每连接监听器工作；剩余的每连接成本就是 socket 写入本身。
