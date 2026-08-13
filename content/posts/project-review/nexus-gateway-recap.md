---
title: "Nexus Gateway：自研 C 语言网关复盘"
date: 2026-08-11
draft: false
categories: ["project-review"]
tags: ["c语言", "网关", "网络编程", "http"]
summary: "从零实现一个 C 语言 HTTP 网关的心路历程：架构设计、技术难点与踩坑记录。"
---

## 项目背景

做这个项目的目的很直接：**系统性地学习 HTTP 网关**。HTTP 网关是网络编程的集大成者——事件驱动 IO、协议解析、代理转发、负载均衡、进程管理、优雅重启，每一项都是后端开发的核心基础。与其零散地看博客，不如从零手写一个，把每个机制真正跑通。

技术选型上用了 C99 + Linux 系统调用，不依赖任何第三方网络库（socket、epoll、sendfile、共享内存全部直接用系统 API）。原因有二：

1. 用框架/库会把底层细节藏起来，学不到内核层面的东西；
2. HTTP 网关性能敏感，C 语言最贴近系统，也最能暴露问题。

项目目标定得很务实：**不追求功能大而全，而是把一条完整的转发链路做深**——从客户端连接到 HTTP 解析、限流防护、上游选择、异步转发、访问日志，再到多进程平滑重启，端到端全部打通。

## 总体架构

整体采用 **Master-Worker 多进程模型**：

```
Master（管理进程）
 ├─ 信号处理：SIGHUP 平滑重启 / SIGCHLD 回收
 ├─ 共享内存：配置双缓冲 + 原子标志
 └─ fork 出 N 个 Worker（默认 = CPU 核数）

Worker（事件驱动进程，互不共享状态）
 ├─ 独立 epoll 实例
 ├─ SO_REUSEPORT 共享监听端口（内核分发连接）
 ├─ HTTP 解析 → 限流/黑名单 → 加权轮询 → 异步转发
 └─ 健康检查线程（每 Worker 一个）

后端集群（upstream，SWRR 加权轮询 + 健康摘除）
```

一次请求的数据流（连接状态机）：

```
accept → CONN_READING_REQ（epoll 等客户端可读）
      → 读请求 + HTTP 状态机解析
      → 黑名单(403) / 令牌桶限流(429) 检查
      → /static 前缀 → 静态文件（sendfile 直发）
      → 其余 → 头部改写（清洗伪造头 + 注入 Req-ID）
      → 非阻塞 connect 上游（CONN_CONNECTING_UPSTREAM，等 EPOLLOUT）
      → getsockopt(SO_ERROR) 确认 → 写请求（CONN_READING_UPSTREAM）
      → 读响应 → 写回客户端 → Nginx 格式访问日志 → 关闭连接
```

核心模块划分：`epoll_loop`（事件循环）、`http_parser`（HTTP 状态机）、`http_rewrite`（代理头）、`upstream`（加权轮询）、`proxy`（上游连接）、`rate_limit`（令牌桶）、`health`（健康检查）、`master`/`worker`（进程模型）、`zerocopy`（零拷贝）。

值得一提的是，项目里实际存在**两层状态机**，各管一件事：

- **连接状态机**（worker.c 的 `conn_state_t`）：管一个请求的完整旅程——读请求 → 连上游 → 读响应 → 回客户端；
- **HTTP 解析器状态机**（http_parser.c 的 `hp_state_t`）：管单个报文的解析进度——请求行 → 头部 → body。

外层状态机停在"读请求"阶段时，调用内层解析器驱动其流转，解析完（HP_DONE）外层才继续。TCP 分包导致的数据不完整，由解析器的**分段喂入**机制兜住——任何状态下都能存进度、等下一片数据。

## 关键技术决策

**1. 事件驱动 vs 多线程 per-connection**

经典的选择题：为每个连接开一个线程，还是事件驱动？前者在 C10K 场景下线程数爆炸、上下文切换开销巨大；事件驱动让一个 worker 用 epoll 管理成千上万个连接，只在有事件时才处理。代价是编程复杂度高——所有逻辑必须写成**非阻塞 + 状态机**：

```c
// worker.c：epoll 事件按连接状态分发
if (c->state == CONN_READING_REQ && (ev & EPOLLIN)) {
    handle_client_read(c);
} else if (c->state == CONN_CONNECTING_UPSTREAM && (ev & EPOLLOUT)) {
    handle_upstream_writable(c);
} else if (c->state == CONN_READING_UPSTREAM && (ev & EPOLLIN)) {
    handle_upstream_readable(c);
}
```

**2. 多进程 vs 多线程**

选多进程：崩溃隔离（worker 挂了 Master 自动 respawn）、各 worker 零共享状态天然无锁、进程替换天然支持平滑重启。代价是进程间通信成本高——用共享内存 + 双缓冲解决（详见踩坑 1）。

**3. SO_REUSEPORT 解决惊群**

多个 worker 如果都 epoll_wait 同一个监听 fd，新连接到来会唤醒所有 worker（惊群）。用 SO_REUSEPORT 让每个 worker 独立 bind+listen 同一端口，**内核按四元组 hash 直接分发**，只有被选中的 worker 被唤醒，从根上规避惊群——这也是 Nginx 1.9+ 的做法。

**4. 手写 HTTP 解析器**

用状态机（请求行 → 头部 → body）而非整包缓冲，支持**分段喂入**，为流式读取做准备；头部转小写 + FNV-1a 哈希检索。自己写解析器能彻底理解 HTTP 协议细节，也方便后续扩展（如 WebSocket 升级握手）。

**5. 零拷贝：sendfile / splice**

静态文件用 sendfile（文件→socket，0 次 CPU 拷贝）；代理转发方向是 socket→socket，传统 sendfile 不支持，用 splice + pipe 中转实现（详见踩坑 3）。

**6. 健康检查：阻塞操作隔离出事件循环**

健康检查的 TCP 探测是阻塞调用（connect 最坏要等满 1 秒超时），放进事件循环会卡死整个 worker，所以单独开一个 **pthread** 周期性探测：5 秒间隔、1 秒超时、连续失败 2 次才摘除节点（防抖动），恢复时一次成功即回归（摘除要谨慎、恢复要果断）。每个 worker 因此是"一个事件循环主线程 + 一个健康检查线程"的结构。

**7. 与内核的分工：哪些是"白给的"，哪些是自己写的**

事件驱动的底层机制都是内核实现的：epoll 的红黑树与就绪链表、事件回调唤醒、非阻塞语义（EAGAIN）、SO_REUSEPORT 分发、信号投递、DMA 零拷贝。代码里真正自己实现的是**机制之上的工程**：

| 内核提供（直接调用） | 自己实现（机制之上的策略） |
|---|---|
| epoll 就绪通知 | 事件分发：这个 fd 是谁的、处于哪个阶段、该调哪个函数 |
| 非阻塞 socket | EAGAIN 处理策略、连接状态机 |
| 信号投递（SIGCHLD/SIGHUP） | handler 只置标志，实际逻辑放主循环 |
| SO_REUSEPORT 内核分发 | 多进程协调：generation、双缓冲、排空检测 |

这也是项目含金量所在：内核只告诉我"哪个 fd 有事"，但"这件事怎么处理、处理完去哪、多进程怎么协调"全部是自己设计的。

## 踩坑记录

### 踩坑 1：平滑重启时新 Worker 差点"自杀"（最有价值）

**现象**：连续两次 SIGHUP 热重启后，第二次拉起的新 worker 刚启动就进入排空模式并退出，服务出现短暂不可用。

**排查**：查 master.log 发现新 worker 的日志显示 `drain_mode=1 (gen=1)`——它启动时共享内存里的 `shutting_down` 标志还是 1（第一次重启设置的，还没被清掉），新 worker 误以为自己也该排空。

**根因**：`shutting_down` 是全局标志，没有区分"这个信号是发给谁的"。新 worker 和老 worker 读的是同一个标志，无法区分信号的新旧。

**修复**：引入 **generation 计数器**。每次重启递增 generation，worker 启动时记录自己的 gen，只有 `当前 gen > 我的 gen` 才响应排空信号：

```c
// worker.c：worker 启动时记录自己的 generation
g_my_generation = atomic_load(g_drain_generation);
// 排空判定：信号必须来自"比我更新的 generation"
int should_drain = current_shutting && (current_generation > g_my_generation);
```

修复后连续多轮热重启全部正常，新 worker 对旧信号天然免疫，实测 `leaked=0`。

### 踩坑 2：非阻塞 connect 的"假失败"

**现象**：转发时上游连接经常失败，或偶发连接建立后请求发不出去。

**排查**：直接调用 connect 后检查返回值 < 0 就报失败。但非阻塞 socket 的 connect 正常会返回 `EINPROGRESS`（连接还在内核里建立），此时**既不是成功也不是失败**。

**根因**：把 EINPROGRESS 当成错误处理了。

**修复**：EINPROGRESS 时把上游 fd 注册 EPOLLOUT，等可写事件后用 `getsockopt(SO_ERROR)` 拿真实结果：

```c
int r = connect(fd, (struct sockaddr*)&addr, sizeof(addr));
if (r < 0 && errno != EINPROGRESS) { close(fd); return -1; }  // 真失败
// 等 EPOLLOUT 事件 → getsockopt(SO_ERROR) 确认
```

### 踩坑 3：sendfile 不支持 socket→socket

**现象**：代理转发想用 sendfile 直接搬数据，调用返回 EINVAL。

**排查**：查 man 手册发现 sendfile 要求 in_fd 必须是支持 mmap 的文件，socket→socket 在老内核上直接报错（Linux 2.6.33+ 部分支持，但受限且不可靠）。

**根因**：零拷贝 API 有方向限制，不是万能胶水。

**修复**：改用 **splice + pipe**——splice 支持任意 fd 但两端至少一个是管道，于是建一个管道做中转：`splice(socket → pipe) + splice(pipe → socket)`，数据全程在内核态流动：

```c
ssize_t nexus_zc_splice(int from_fd, int to_fd, size_t len) {
    int pipefd[2];
    pipe(pipefd);
    while (len > 0) {
        ssize_t n = splice(from_fd, NULL, pipefd[1], NULL, len, 0);
        if (n <= 0) break;
        ssize_t m = splice(pipefd[0], NULL, to_fd, NULL, n, 0);
        if (m <= 0) break;
        len -= m;
    }
    close(pipefd[0]); close(pipefd[1]);
    return total;
}
```

### 踩坑 4：QPS 上不去，瓶颈不在解析而在连接管理

**现象**：wrk 压测 1KB 小包只有 650 QPS，p99 高达 1.41s，和预期差很远。

**排查**：strace 看系统调用频率、ss 看连接状态，定位到三个问题：① 每请求一次完整建连/断连（无 keep-alive），TIME_WAIT 大量堆积；② 每请求同步写一次日志（锁 + fprintf + flush）；③ 连接池 1024 槽位线性扫描 + 并发超限直接失败，wrk 重试加剧拥塞。

**根因**：正确性优先的设计（每请求一连接、同步日志）在低并发下没问题，高并发下全变成瓶颈。

**修复方向**：优先级排序——keep-alive + 上游连接池 → 异步日志 → 连接池哈希索引 → ET 模式循环读写。这次踩坑最大的收获是理解了**"先正确再性能"必须配套性能意识，否则正确性设计会成为性能天花板**。

补充一点：连接复用会引入新的复杂度，不是白拿的收益——空闲连接必须超时踢掉，否则 fd 无限堆积（当前代码没有任何超时机制）；下一个请求可能分片到达，解析器要能从"上一请求解析完"平滑过渡到"等下一个请求行"（分段喂入的状态机天然支持）；响应要能判断"发完了"的边界（Content-Length / chunked），不能像现在这样读一次就关。

## 性能与效果

压测环境：Ubuntu 24.04 / 12 核 / 8G，wrk -t8 -c1000 -d10s。

| 场景 | 指标 | 备注 |
|---|---|---|
| 代理转发（1KB 响应） | 650 QPS | p99 1.41s，瓶颈已定位（见踩坑 4） |
| 静态文件（50MB） | 2.81 GB/s 吞吐 | 顺序大块读写 + 页缓存命中；零拷贝因环境限制降级 read+write |

650 QPS 是在 1000 并发下测得的——连接池 1024 槽位已接近打满，属于可解释的正常范围而非异常劣化；真正的痛点（p99 1.41s 的长尾）来自排队与重试，根因见踩坑 4。

与 Nginx 相比：

- **机制层面**：epoll 事件驱动、Master-Worker、平滑重启、SWRR 加权轮询的实现思路与 Nginx 同源；
- **工程层面**：差距明显——Nginx 有模块体系、按需连接池、异步日志、完整测试与配置校验体系。

已知局限：代理路径响应缓冲 8KB，大响应会截断（静态文件路径不受影响，sendfile 循环发送）；上游全部不可用时直接断连而非返回 503；限流表满时 fail-open。这些都有明确的改进方案，优先级已排好。

## 复盘与收获

**1. 对 HTTP 网关的理解从"概念"变成"机制"**

以前知道"网关要支持高并发"，做完之后能说清每一步为什么：epoll 为什么比 select 快（红黑树注册 + 就绪链表回调）、为什么必须非阻塞（事件循环不能被单条连接卡死）、多进程为什么无锁（零共享状态）、SO_REUSEPORT 为什么能防惊群（内核级分发）。

对**事件驱动**本质的理解也发生了质变：它是控制流反转——程序不主动读数据，而是内核通知"有数据了"才处理；每个连接的处理都是一小段非阻塞代码，做完当前能做的就回到 epoll_wait 让出控制权，进度显式存在连接对象里。相当于**用状态机字段手动实现了线程的"挂起/恢复"**，换来的是单线程管理上万连接且零切换开销；而整个系统只有一个睡眠点（epoll_wait），连接本身从不"睡"，只是挂在红黑树上等待被叫号。

**2. 最大的成长在"排障"而不在"写代码"**

踩坑 1 和踩坑 2 都是时序/状态类问题，靠日志 + 状态机推理定位，而不是靠猜。这也让我养成了习惯：**先想清楚状态转换是否完备，再写代码**。

**3. 工程意识：正确性、性能、可观测性三者要同时考虑**

早期只关注"转发通不通"，压测后才意识到连接管理策略、日志写入方式就是性能本身；而 Req-ID、Nginx 格式访问日志这些"可观测性"设计，在后端排障时价值极大——通过同一个 Req-ID 可以把网关日志和业务日志串成完整链路。

**4. 后续方向**：keep-alive 与上游连接池、异步日志、响应流式转发（修掉 8KB 截断）、统一健康检查进程、HTTPS 支持（异步 TLS 握手）。

---
