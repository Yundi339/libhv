# 项目分支相对原版 libhv 的改动

本文记录 Intertwine 使用的 project/main 分支在原版 libhv 基础上增加的改动，重点是 HTTP 静态文件缓存和发送路径。

## 对比口径

- 原版基线：ithewei/libhv 的共同祖先 9bfb2b3，版本描述为 v1.3.4-28-g9bfb2b3。
- 项目分支：Yundi339/libhv 的 project/main，记录时的头提交为 e9e9f93。
- 对比命令：git diff upstream/master...project/main。
- 当前净差异：8 个文件，453 行新增，106 行删除。

上游 master 在基线之后继续演进，本文只记录 project/main 相对共同基线新增的部分，不把上游已经独立实现的改动算作本项目分支的新增能力。

## 总览

| 模块 | 主要改动 | 目的 |
| --- | --- | --- |
| http/server/FileCache | 可配置的头部预留和缓存上限、每条目互斥锁、不可变发布语义、同路径加载合并 | 修复并发刷新问题，降低大响应头和大文件对缓存的干扰 |
| http/server/HttpHandler | 响应头过大时拆分发送、发送期间持有缓存条目、请求头早期拒绝立即响应 | 避免 buffer 被并发替换，改善大请求和异常请求行为 |
| 测试与构建 | 新增 FileCache 单元测试，并接入 Make 和 CMake | 覆盖刷新、失败清理、并发加载和头部回退场景 |

## FileCache 增强

原版 FileCache 使用固定的 HTTP_HEADER_MAX_LENGTH = 1024 和 FILE_CACHE_MAX_SIZE = 4 MiB。项目分支改为每个缓存实例可配置：

    FileCache cache;
    cache.SetMaxHeaderLength(8192);
    cache.SetMaxFileSize(16 * 1024 * 1024);

默认值变为：

- FILE_CACHE_DEFAULT_HEADER_LENGTH：4096 字节。
- FILE_CACHE_DEFAULT_MAX_NUM：100 条。
- FILE_CACHE_DEFAULT_MAX_FILE_SIZE：4 MiB。

头部预留空间默认从 1 KiB 提高到 4 KiB。对常见静态文件响应，这减少了头部写不进预留区的回退路径；代价是每条缓存最多多保留约 3 KiB 内存。

### 并发与生命周期

原版实现会在请求仍然持有缓存条目时刷新同一条目，resize_buf 可能移动底层 buffer，导致旧指针读到已释放或重新解释过的内存。项目分支做了以下调整：

1. 已发布条目的文件内容视图不再原地重写。检测到文件变化时，先在缓存外构造完整的新 file_cache_t，读取成功后再替换 LRU 中的条目；stat 统计字段仍受条目锁保护。
2. file_cache_t 增加内部互斥锁。SendHttpResponse 持有共享指针和条目锁，直到数据已同步写入 libhv IO 队列，避免发送过程中条目被淘汰或 buffer 被重写。
3. Open() 增加同路径访问控制。同一个路径的并发冷加载会等待同一个加载过程并复用同一份结果；不同路径仍然可以并发加载。
4. 过期清理使用 try_to_lock。正在发送或刷新的条目不会阻塞清理线程，本轮跳过后由下一轮处理，避免与 Open() 形成锁顺序问题。

### 数据一致性与失败处理

原版主要通过 mtime 判断文件是否变化，并且部分失败路径会保留旧条目。项目分支补充了：

- 对比 mtime、size 和 mode，减少只改属性或内容但时间戳异常时的误判。
- Windows 路径也检查 mtime、size 和 mode，并且不再把目录伪装成 fd = 0。
- 文件读取循环处理短读、EINTR 和意外 EOF。
- stat、open、读取或超限失败时移除旧缓存条目，避免继续返回旧内容。

### 头部与 buffer 边界

file_cache_t 的 buffer 明确拆成三个概念：

- header_reserve：文件内容前的预留空间。
- header_used：实际写入响应头的字节数。
- filebuf：文件内容区域。

prepend_header() 从原来的 void 改为返回 bool。头部超过预留区时不再静默失败，而是标记回退并让 HttpHandler 先发送响应头，再发送文件内容。重新分配 buffer 后也会主动清空旧的 httpbuf 视图。

## HTTP Handler 集成

### 大响应头回退

静态文件响应头超过缓存预留区时，项目分支不会继续使用未拼接成功的 buffer，而是先从响应对象发送头部，再从 filebuf 发送文件体。这样不会复制一份额外文件内容，也能避免原实现里头部被丢弃或指针状态不一致的问题。

### 发送期间的数据保护

SendHttpResponse() 生成发送 buffer 前会保留 file_cache_ptr 并锁定条目。libhv 的 hio_write() 会把数据复制进 IO 队列后才返回。锁在写操作完成后释放，保证发送过程中不会被另一个线程刷新或关闭缓存。

### Range 请求不进小文件缓存

原版对 HEAD 和 Range 请求都不读取文件内容到 FileCache。项目分支调整为只有 HEAD 不读取；Range 请求走大文件发送路径。这样可以避免大文件因为一个 Range 请求被读入或污染缓存。

### 请求头早期拒绝

请求头校验失败时立即发送错误响应并关闭 keep-alive，不再继续等待请求体。对超大请求体或恶意请求，这能更早释放连接和内存。

## 与 HTTP 服务配置联动

HttpService::max_file_cache_size 会同步到每个 HTTP loop 的 FileCache::max_file_size。发送完成后，超过当前服务配置上限的条目会从缓存移除。原实现使用编译期硬编码上限，服务配置调大后行为不一致。

## 测试覆盖

新增 unittest/file_cache_test.cpp，覆盖：

- 文件刷新后发布新条目，旧的 in-flight 共享指针内容不变。
- 文件删除或刷新失败后立即失效，不再等旧 stat 周期。
- SetMaxFileSize() 与单次请求 max_read 覆盖的关系。
- 超大响应头回退到文件内容视图。
- 同一路径并发冷加载只产生一个发布条目。
- 并发刷新后所有新请求拿到同一份新数据，旧请求仍持有旧数据。

测试已接入：

- make unittest
- scripts/unittest.sh
- CMake unittest target

## 兼容性影响

### 源码 API

主要公共类型仍是 FileCache、file_cache_t 和 file_cache_ptr，现有基础用法可以继续编译。但有以下需要注意的变化：

- prepend_header() 返回值从 void 变为 bool，调用方应检查返回值。
- resize_buf() 增加两参数版本；原单参数版本保留。
- FileCache::OpenParam::max_read 的默认值从固定宏改为 0，表示使用实例级 max_file_size。
- 旧的 HTTP_HEADER_MAX_LENGTH、FILE_CACHE_MAX_NUM、FILE_CACHE_MAX_SIZE 宏被默认值宏替代，外部代码如果直接引用旧宏需要调整。

### 行为兼容性

已发布的缓存条目不会再被原地刷新，而是被替换为新对象。依赖同一个路径永远返回同一个 file_cache_t 指针的代码需要改为按共享指针使用。

## 上游同步状态

本文记录的是 project/main 的自有增强。截至 2026-11-26，上游 ithewei/libhv master 已经有 36 个提交不在 project/main 历史中，其中包含 FileCache、HTTP/2、代理、DNS、WebSocket 和构建系统的后续修复。定期升级 project/main 时，应先对比上游最新变更，再保留这里针对 Intertwine 场景的 FileCache 语义。
