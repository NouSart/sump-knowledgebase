# 02｜WebSocket 与消息协议：一封委托单怎样找到自己的结果

<!-- attribution:start -->
> 由 **xml-nya-meow** 编写与整理，最初用于个人学习与知识积累。如有疏漏、错误或不清楚之处，欢迎指出。
>
> 本页围绕 [SUMP 主项目](https://github.com/NouSart/sump) 编写，为个人学习资料；项目源码与引用内容保留原作者及贡献者归属。
<!-- attribution:end -->

**对应版本：**2026-10-26，提交 `149451dbf9ec953006da7c1926c336cfd9da0633`。

**阅读依据：**本轮只核对源码与测试定义。示例是教学模拟，不是抓取的真实网络消息，也不代表本轮已执行测试。

## 1. 这一章学会什么

你会学会分清“连接方式”“消息格式”“具体动作”，以及 `id`、`trace`、`boss` 三个容易混淆的字段。

先想象快递：道路负责把车送到；信封负责写地址；包裹里装具体东西。这三层不能混为一谈。

## 2. 先读懂一个地址

以下是本机计时器的默认地址示例：

```text
ws://127.0.0.1:9801/ws
```

| 这一段 | 表示什么 | 白话理解 |
|---|---|---|
| `ws` | 使用 WebSocket | 连接建立后，双方可持续发送消息 |
| `127.0.0.1` | 当前机器自己 | 宿主机与虚拟机各有自己的“自己” |
| `9801` | 端口号 | 同一台机器上不同程序的接待窗口 |
| `/ws` | HTTP 路径 | 程序把这个入口升级为 WebSocket |

WebSocket 是通信方式，不规定业务消息必须是 JSON。SUMP 原生服务使用它发送 MessagePack 编码的信封。

**长连接能力**表示连接可以保持着；不等于应用一定复用连接。当前 `service.Call` 每次建立一条连接，得到响应后关闭。到设置中心的注册连接则保持，用来接收名册等事件。

## 3. JSON 和 MessagePack：同一份数据的不同装法

JSON 常见形状是 `{"ms":3000}`，人能直接读。MessagePack 把整数、文字、列表等编码为二进制字节，便于程序传输。

**序列化**就是把程序里的数据装成可传输的形式；**反序列化**就是收件后按约定拆出来。

SUMP 使用 `protocol.Marshal` 编码整个信封，再放进 WebSocket 的二进制消息中。普通文本形式的 JSON 不能直接粘到原生 WS 入口，就期望被当成同一协议。

二进制不是加密。把内容编码成 MessagePack，并不能防止能读取网络数据的人理解它。

跨语言时还要核对数据类型。比如 Go 的整数与 JSON 解码后的浮点数并不天然相同；“看起来都是 3000”不等于内部类型一致。项目在 `protocol/payloads.go` 提供 `NormalizeJSONNumbers`，并用 `jsonnum_test.go` 检查转换。

## 4. 外层信封与内层正文

下面用 JSON 排版展示一条消息，便于阅读。它不是可以直接发送的原生 WS 文本。

```json
{
  "v": 1,
  "id": "req-A",
  "trace": "chain-A",
  "boss": "reasoner",
  "type": "jump",
  "from": "reasoner",
  "to": "timer",
  "payload": {
    "action": "sleep",
    "input": {"ms": 3000, "note": "练习"}
  }
}
```

`req-A` 和 `chain-A` 是教学标签。真实 `NewEnvelope` 会生成消息编号；没提供 trace 时也会生成 trace。编号生成不需要模型来猜。

| 字段 | 白话解释 | 这一例里的作用 |
|---|---|---|
| `v` | 信封格式的版本 | 接收方按第 1 版结构理解 |
| `id` | 这一张委托单的编号 | 找回与 `req-A` 对应的响应 |
| `trace` | 整条处理链的编号 | 后续不同请求仍可沿用 `chain-A` |
| `boss` | 本跳面向谁工作、结果归属提示 | 此例显式填 `reasoner` |
| `from` / `to` | 发件服务与收件服务 | reasoner 请求 timer |
| `type` | 外层消息类型 | `jump` 表示访问某个动作 |
| `payload.action` | 服务里的具体动作 | `sleep` |
| `payload.input` | 动作参数 | 等待时长与备注 |

**payload** 意思是载荷，也就是信封装的正文。`jump` 与 `sleep` 属于不同层：前者告诉骨架怎么分发，后者告诉计时器做什么。

## 5. id、trace、boss 分别回答不同问题

`id` 回答“这是谁那一次请求的回包”；`trace` 回答“它属于哪条工作链”；`boss` 表达“这一跳为谁工作”。

例如 A 请 B 办事，B 又请 C 帮忙。B→C 产生新的请求 id，但可以沿用 trace。B 可以明确选择本跳的 boss，不是机械继承 A 的权限。

**boss 不是万能回邮地址。**最终发给谁，要读动作自身的约定。timer 的到点补报使用原调用方；门户倒计时展示另外携带 boss 信息。不能仅凭字段名字推断每个服务的路由。

`from` 和 `boss` 也不是密码。当前原生 WS 骨架没有统一身份认证，填写一个服务名本身不能证明发送者就是那个服务。

## 6. response 如何返回

`NewResponse` 实际做的核心事情，是交换发件与收件方向，保留原请求的 `id` 与 `trace`。**它不会自动复制请求的 boss。**

响应正文统一是 `ok`、可选的 `data`、可选的 `error`。计时器受理后的示意如下，省略外层信封：

```json
{
  "ok": true,
  "data": {"item_id": "timer-X", "phase": "queued", "fire_at": "2026-10-26T10:00:03+08:00"}
}
```

`item_id` 是异步工作的句柄，类似取件码；它不是原请求 id。示例中的时间只用于说明格式。

`ok:true` 在这里表示受理成功，不能解释为 3 秒已经过去。真正完成由后来的 `report_result` 描述，补报需要保留 `item_id` 才能关联。

## 7. 两个请求交错回来，会不会串线

以下是教学模拟，假设同一个 `protocol.Client` 同时有两个请求：

| 时刻 | 事件 | 客户端怎么做 |
|---|---|---|
| 1 | 发送 id 为 A 的请求 | 在 `pending` 表登记 A 的等待通道 |
| 2 | 发送 id 为 B 的请求 | 再登记 B 的等待通道 |
| 3 | B 的 response 先到 | 查 id=B，把它交给等 B 的人 |
| 4 | A 的 response 后到 | 查 id=A，把它交给等 A 的人 |

`pending` 是程序里的表，不是数据库。每个等待通道只服务对应请求；网络发送使用写锁，避免多个执行流程同时写坏连接。

普通 response 的配对发生在 `protocol.Client`。异步 `report_result` 的归属则由上层业务继续管理，因为最初请求可能早已返回、原连接也已关闭。

所以“WS 客户端会配对 id”不能代替“Reasoner 或 MCP 正确关联晚到的异步结果”。这两件事必须分别实现。

## 8. 超时、错误与大小限制

**超时**表示等到期限还没有拿到结果，不表示对方没执行。付款已经发生但回执丢失，就是最容易理解的例子。

本版 `service.Call` 只对拨号未成功的情况做有界重试；消息发送后出现超时等错误，不会自动再次执行。它避免把未知结果的操作悄悄重做一遍。

**幂等**指重复请求不会产生额外效果。例如“把状态设为关闭”通常可以；“再增加一个计时器”不应随意当作幂等。

`MaxMessageBytes` 可以限制接收的一条 WS 消息大小。正数才启用；零或负数表示没有启用这个限制。超限会关闭连接，不会返回一个完整的业务错误对象。

它限制的是接收方向，并非把任意长输出自动截成短文。业务输出限额仍需要各服务另行处理。

## 9. 动手练习：一眼辨认字段

### 练习 A：只读源码

1. 打开 `protocol/envelope.go`，搜索 `NewResponse`。
2. 找出哪一行把请求 id 交给响应。
3. 查看这个函数是否给 `Boss` 赋值。
4. 打开 `protocol/client.go`，搜索 `pending[env.ID]`，找出登记和取回位置。

**应有结果：**你找到 `env.ID = req.ID`；NewResponse 没有复制 Boss；响应按照 id 查找等待方。

如果误把 trace 当配对键，两个属于同一链的并发请求就不易区分。这个练习的目标是能说清两种编号为什么都存在。

### 练习 B：可选地运行一个纯编解码测试

电脑有满足仓库要求的 Go 时，在源码的 `protocol` 目录打开终端：

```text
go test -run '^TestEnvelopeRoundTrip$' -count=1 -v
```

**预期结果：**显示该测试的 `PASS`。它先编码名片信封，再解码并核对字段，不启动整套 SUMP，也不调用模型。

测试失败时先看第一条实际报错；依赖下载失败、Go 版本不足，与断言失败是不同问题。本章未替你执行这个命令。

## 10. 下一章与源码入口

接下来读 [03-注册发现与配置](03-注册发现与配置.md)：这些地址从哪里来，配置又怎样真正生效。

固定版本源码：[信封](https://github.com/NouSart/sump/blob/149451dbf9ec953006da7c1926c336cfd9da0633/protocol/envelope.go)、[业务载荷与数字转换](https://github.com/NouSart/sump/blob/149451dbf9ec953006da7c1926c336cfd9da0633/protocol/payloads.go)、[请求配对](https://github.com/NouSart/sump/blob/149451dbf9ec953006da7c1926c336cfd9da0633/protocol/client.go)、[编解码测试](https://github.com/NouSart/sump/blob/149451dbf9ec953006da7c1926c336cfd9da0633/protocol/envelope_test.go)、[数字转换测试](https://github.com/NouSart/sump/blob/149451dbf9ec953006da7c1926c336cfd9da0633/protocol/jsonnum_test.go)、[服务调用与限制](https://github.com/NouSart/sump/blob/149451dbf9ec953006da7c1926c336cfd9da0633/service/service.go)。

<!-- attribution-footer:start -->
---

**编写与整理：xml-nya-meow** · 原用途：个人向知识库。

如有不足，欢迎提出勘误或改进建议：[反馈入口](https://github.com/xml-nya-meow/sump-knowledgebase/issues)。

[个人知识库](https://github.com/xml-nya-meow/sump-knowledgebase) · [SUMP 主项目](https://github.com/NouSart/sump)

阅读时请核对本页对应的源码版本；本文不代表 SUMP 官方文档。转载或同步时请保留编写者、来源和适用的许可说明。

内容责任与立场说明：若知识库内容存在问题，由对应编者负责核查与修订；本页内容不代表其他团队成员、组织成员或组织的意见与观点。
<!-- attribution-footer:end -->
