# Agent Note: JSONL 列举改用 stat 探测并备忘录头部行

Status: implemented

[English](2026-08-20-jsonl-list-artifacts-memoization.md) | 中文

## 问题

一个饱和的 `dsh web` 宿主在多个 agent（智能体）并发流式输出时，把列举类 RPC（`subagent.list`、`session.list`）从约 0.5 秒拖到约 100 秒；该饱和在 CPU 侧的另一半——事件流上按连接倍增的扇出——另行处理。扇出共享消除了乘法，但 JSONL 后端的 `listArtifacts` 在构造上依然昂贵：每个 session 目录要串行 await 两次 open-close 存在探测加一次完整头部读取（open、读 8 KB、解压、close）——每个 session 五到七次串行 fs 往返，遍历所有 project 目录，115 个 session 的存储约 600 次 await 的操作。事件循环接近满载时每次往返都要排在循环工作之后，离线约 250 毫秒的扫描就变成数十秒。

## 决策

**stat 探测替代 open 探测。** `probeSessionDir` 以 bigint 身份 stat 反编码产物与主产物。缺失路径保留 `exists()` 语义：ENOENT 仍会执行「父目录允许缺失」守卫，被阻塞的 session 目录因此仍是存储故障而非静默缺失。反编码探测仍然抛出编码不匹配；主探测的身份则喂给备忘录。

**头部备忘录按产物路径键控，stat 身份（dev、ino）匹配即有效。** append-only 日志的首帧不可变：追加和截尾式崩溃修复都不改写第 0 帧，物化发布的是全新 inode。因此热列举完全不做日志读取；同路径被替换的文件（新 inode）重读一次。备忘录有意不设上限——每个存储中的 session 一行头部，上界是后端自身从不清理的盘上存量。

**有界并行探测。** session 目录由 `LIST_PROBE_CONCURRENCY = 16` 个 worker 探测，这是内部调度常量，与 subagent 列举的 `COLD_READ_CONCURRENCY = 4` 同族。结果按目录索引落位，产物顺序与重复 id 拒绝保持确定性；多个目录并发失败时哪个错误先浮出不保证顺序。`listArtifacts` 的返回形状（`{ header, path }`）不变——`listSnapshots` 保留自己的发现后 stat，它兼任发现与快照之间的外部删除守卫。

## 后果

N 个 session 的热列举成本约为 5 次 `readdir` + 2N 次 `stat` 加未命中。在慢速磁盘上的 167 个 session 存储上背靠背实测：此前每次列举 576–725 毫秒；之后首次列举 254 毫秒，热列举 54–86 毫秒——稳态扫描消失。inode 号复用不会跨路径错配头部（键包含路径），同路径重建以新 inode 到达而未命中。两个列举测试随行为更新：头部读取取消测试不再先热身（热列举不读任何东西，无从取消），坏帧列举断言匹配任一坏帧（并行探测的首错不确定）。新增 spec 直接钉住备忘录约定：热列举执行零次 `FileHandle` 读取；同路径 rename 换入的新文件列举出新头部。

## 考虑过的替代方案

- **由写入方维护的持久头部索引**——列举 O(1)，但要新的盘上格式加跨进程协调；预发布立场允许改格式，但备忘录两者都不需要就达到了热的 O(stat)。
- **(size, mtime) 校验**——追加会同时抬高两者，活跃写入的 session 永远命不中；(dev, ino) 才是对第 0 帧真正成立的不变量。
- **只做并行化**——改善冷列举，但热列举每次仍要付完整头部读取。
