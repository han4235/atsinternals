# HostDB 子系统

在 ATS 的设计里，使用 HostDB 子系统作为 DNS 解析结果的缓存，当上层协议状态机需要对 DNS 解析时，先向 HostDB 发起查询请求，而 HostDB 作为 DNS 解析结果的缓存，可以极大的提升 DNS 解析的效率。

我们知道 DNS 系统的设计是允许并希望用户自己建立 DNS Cache Server，以加速 DNS 解析的并降低根服务器的压力，通常 ISP 提供商也会向自己的用户提供公共的 DNS 缓存服务。

HostDB 子系统就相当于是 ATS 系统内部的 DNS Cache Server，它完全模拟并实现了 DNS 缓存服务的功能：

- 接收 DNS 解析请求（通过 HostDBProcessor 提供的接口）
- 查询内部的 DNS 结果缓存
- 对 DNS 结果缓存进行持久化保存，即使 ATS Crash 也可以确保缓存结果不丢失
- 当在缓存中找不到结果时，或者缓存过期时，会通过 DNS 子系统进行查询（通过 DNSProcessor 提供的接口）

在 HttpSM 进行 DNS 查询时，是通过直接调用 HostDB 的查询接口，对指定的域名进行地址解析，当 HostDB 发现它的缓存无法找到这个域名的解析结果时，会自动调用 DNS 子系统提供的地址解析接口，发起 DNS 请求。

请注意 HostDB 与 DNS 子系统的层级关系的设计：

```
  +--------+        +--------+        +--------+                 +----------+
  |        |        |        |        |        |                 |          |
  |        +-------->        +-------->        +==== UDP/TCP ====>          |
  |        |        |        |        |        |                 |    DNS   |
  | HttpSM |        | HostDB |        |   DNS  |                 |          |
  |        |        |        |        |        |                 |  Server  |
  |        <--------+        <--------+        <==== UDP/TCP ====+          |
  |        |        |        |        |        |                 |          |
  +--------+        +--------+        +--------+                 +----------+
```

这是一种逐层向下发起调用，然后再逐层回调的逻辑，这是 ATS 里堆叠多个子系统在一起的典型设计。

其它两种不是很合理的设计模式：

1. 如果把 HostDB 看成是 DNS 的子系统，
	- HostDB 对 HttpSM 不可见，HttpSM 直接调用 DNS 的接口，那么这个设计就会变的复杂，
	- 当 DNS 得到解析结果时，要同时向 HostDB 和 HttpSM 回调以传递解析结果，这无疑增加了 DNS 的复杂度
	- 可以这么做，但是不推荐
2. 如果让 HttpSM 首先查询 HostDB，查询不到再去直接调用 DNS 的接口
   - 那么 HttpSM 还需要负责将 DNS 解析的结果再送给 HostDB 进行缓存
   - 这样就增加了 HttpSM 的工作，而且这个工作很显然应该是属于 DNS 结果的处理，不应该由 HttpSM 负责
   - 非常不合理的设计

如果我们将来需要设计这种需要多个子系统配合的功能时，请参考 DNS 与 HostDB 的设计模式。

另外，为了实现反向代理等功能，社区对 HostDB 的功能进行了相应的扩展。


