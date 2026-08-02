# DN42 内部 iBGP Deploy 设计文档

日期：2026-08-02

## 目标

在 `/workspace/dn42` 已完成的：

- 内部网络领域模型
- 内部 `WireGuard` deploy 全链路
- 内部 `OSPF` deploy 主路径

基础上，继续补齐内部控制平面的下一阶段设计：`iBGP deploy`。

本设计覆盖：

1. 内部 `iBGP` 中间层对象
2. 内部 `iBGP` deploy payload
3. 节点侧 `ibgp.internal.deploy` / `ibgp.internal.remove` / `ibgp.internal.status`
4. 后端状态回写与错误处理

## 目标边界

本阶段的目标不是实现完整多协议编排，而是把“内部 `iBGP` 如何独立于传输层和 `OSPF` 层完成下发”这条链路设计清楚。

### 本设计要解决的问题

- 后端如何从 `RoutingSession(kind="ibgp")` 派生出可部署的 `iBGP` 视图
- 如何让 `iBGP` deploy 与现有 `internal WireGuard deploy`、`OSPF deploy` 保持独立
- 节点如何接收并执行内部 `iBGP` 配置
- 后端如何回写 `RoutingSession.oper_state`
- 如何为后续：
  - `ZeroTier` 承载
  - route-reflector
  - 多协议编排器

保留清晰边界

### 本设计不解决的问题

- `ZeroTier` 承载上的 `iBGP`
- route-reflector / route-server
- 复杂路由策略编排
- 多协议联合事务回滚
- 内部网络 UI

## 为什么不能把 iBGP 塞回 OSPF deploy

当前 `OSPF` 已承担内部 underlay 协议层职责：

- 消费已存在的内部接口
- 配置 IGP 邻接
- 回写 `RoutingSession(kind="ospf").oper_state`

而 `iBGP` 的职责不同：

- 它属于 overlay 控制平面
- 它依赖 underlay reachability，但不负责建立 underlay
- 它的成功或失败应只反映到 `RoutingSession(kind="ibgp")`

因此本设计明确：

- `internal.*`
  - 只负责内部传输层
- `ospf.internal.*`
  - 只负责内部 IGP
- `ibgp.internal.*`
  - 只负责内部 BGP 控制平面

三者并存，但不混用。

## 总体方案

本设计采用：

### `RoutingSession(kind="ibgp") + InternalFabricRenderContext -> InternalIbgpNodeSpec -> InternalIbgpDeployPayload -> ibgp.internal.deploy`

也就是说：

1. 领域层提供 `InternalFabricRenderContext`
2. 查询出 `RoutingSession(kind="ibgp")`
3. 结合当前节点能力、地址和承载信息构建 `InternalIbgpNodeSpec`
4. 派生 `InternalIbgpDeployPayload`
5. 后端通过 WSS 发送 `ibgp.internal.deploy`
6. 节点写入 `iBGP` 配置并执行 `birdc configure`
7. 返回执行结果
8. 后端更新 `RoutingSession.oper_state`

## 第一版依赖关系

当前第一版 `iBGP deploy` 明确依赖：

- 已存在的 `WireGuard` 传输层
- 已存在的内部地址和 `router_id`
- 已存在的 `OSPF` 主路径

这意味着：

- `iBGP` 不负责建立隧道
- `iBGP` 也不负责创建 `OSPF` 接口或补路由
- 它只配置 BIRD 中的内部 `BGP` 邻居

## 核心对象

## 1. InternalIbgpPeerSpec

表示某个节点视角下的一条内部 `iBGP` 邻居配置。

建议字段：

- `routing_session_id`
- `peer_node_id`
- `peer_node_name`
- `transport_kind`
- `local_asn`
- `peer_asn`
- `local_address`
- `peer_address`
- `topology_role`

说明：

- `routing_session_id`
  - 用于状态回写精确定位 `RoutingSession`
- `transport_kind`
  - 当前第一版只允许 `wireguard`
- `local_asn`
  - 当前节点内部 ASN
- `peer_asn`
  - 对端节点内部 ASN

## 2. InternalIbgpNodeSpec

表示某个节点应接收的一整份 `iBGP` 配置视图。

建议字段：

- `node_id`
- `node_name`
- `router_id`
- `transport_kind`
- `peers`
- `render_version`

说明：

- `transport_kind`
  - 当前一份 `NodeSpec` 只对应一种承载分支
- `peers`
  - 该节点上的所有 `iBGP` 邻居
- `render_version`
  - 用于调试和后续兼容

## 3. InternalIbgpDeployPayload

表示后端一次发送给节点的 `iBGP` 下发请求。

建议字段：

- `request_id`
- `node`
- `transport_kind`
- `config_name`
- `render_version`
- `bird_config`
- `peers`

说明：

- `config_name`
  - 节点本地 `iBGP` 配置文件基名
- `bird_config`
  - 最终渲染完成的内部 `iBGP` 配置文本
- `peers`
  - 结构化邻居信息，仅用于诊断/未来扩展

## 与现有模型的关系

这次 `iBGP` deploy 以现有内部网络模型为基础：

- `TransportLink`
  - 决定底层承载类型
- `RoutingSession`
  - 只有 `kind="ibgp"` 的进入本链路
- `NodeCapability`
  - 约束节点是否支持 `iBGP`

### 基本筛选规则

一条 `RoutingSession(kind="ibgp")` 进入 `iBGP` deploy 链路前，至少应满足：

- 关联的 `TransportLink` 存在
- `RoutingSession.admin_state = enabled`
- 关联两端节点都 `supports_ibgp = true`
- 两端节点都拥有 `internal_asn`
- 两端节点都拥有 `internal_router_id`
- `TransportLink.admin_state = enabled`

## 第一版承载范围

虽然整体上 `iBGP` deploy 未来可能支持多种承载，但当前第一版正式支持的是：

- `transport_kind = wireguard`

同时明确不做：

- `transport_kind = zerotier`

这部分会留给后续 `ZeroTier` 子项目单独设计和实现。

## WireGuard 分支的设计

当前第一版 `iBGP` deploy 的主路径是：

- `WireGuard` 作为内部承载

也就是说：

1. `WireGuard` 隧道已经存在
2. `OSPF` 已负责内部 reachability
3. `iBGP` 层从现有内部地址与节点能力中提取：
   - 本地 ASN
   - 对端 ASN
   - 本地地址
   - 对端地址
4. `iBGP` 层只负责渲染 BIRD `BGP` 邻居配置

这里的关键边界是：

- `WireGuard` 层拥有接口生命周期
- `OSPF` 层拥有 IGP 生命周期
- `iBGP` 层拥有控制平面邻居生命周期

## 配置文本与命名规则

内部 `iBGP` deploy 不复用 `OSPF` 或 `WireGuard` 的 `config_name`。

建议新增：

- `config_name`

规则建议：

- `IBGPI_<transport>_<node_suffix>`

例如：

- `IBGPI_wireguard_tokyo`

约束：

- 必须满足安全名称规则
- 必须稳定
- 必须与 `internal WireGuard`、`OSPF` 配置名分离

节点上的落盘路径建议为：

- `/etc/bird-internal/<config_name>.conf`

说明：

- 当前内部 `OSPF` 已建议使用 `bird_internal_dir`
- `iBGP` 可以继续共用这个内部协议目录
- 但应通过独立 `config_name` 和独立命令空间保持协议层边界

## 节点协议设计

## 1. 新增命令

节点 WSS 新增：

- `ibgp.internal.deploy`
- `ibgp.internal.remove`
- `ibgp.internal.status`

其中本阶段重点是：

- `ibgp.internal.deploy`

## 2. ibgp.internal.deploy 请求体

建议请求体：

```json
{
  "request_id": "6f67e8c1-8241-4d1a-a0da-2f4f9b79f3e9",
  "node": "tokyo",
  "transport_kind": "wireguard",
  "config_name": "IBGPI_wireguard_tokyo",
  "render_version": "internal-ibgp-v1",
  "bird_config": "# Generated internal iBGP config...\nprotocol bgp IBGPI_wireguard_tokyo_osaka {\n  local as 4242420001;\n  neighbor fd42::2 as 4242420001;\n  source address fd42::1;\n}\n",
  "peers": [
    {
      "routing_session_id": "session-1",
      "peer_node_id": "node-b",
      "peer_node_name": "osaka",
      "transport_kind": "wireguard",
      "local_asn": "4242420001",
      "peer_asn": "4242420001",
      "local_address": "fd42::1",
      "peer_address": "fd42::2"
    }
  ]
}
```

## 3. 字段校验

建议节点侧校验：

- `request_id`
  - 必填
  - 非空
  - 最大长度 64
- `node`
  - 必填
  - 最大长度 64
  - 不允许 `/`、`\`
- `transport_kind`
  - 必填
  - 当前实现只接受 `wireguard`
- `config_name`
  - 必填
  - 必须匹配安全名称规则
- `render_version`
  - 必填
  - 最大长度 64
- `bird_config`
  - 必填
  - 最大 32 KiB
  - 不允许 NUL
- `peers`
  - 可为空数组
  - 若存在，每个元素必须是合法对象

## 4. ibgp.internal.deploy 返回体

建议返回体：

```json
{
  "ok": true,
  "applied": true,
  "output": "deployed internal ibgp config\nbird reload:\n...",
  "files": [
    "/etc/bird-internal/IBGPI_wireguard_tokyo.conf"
  ],
  "config_name": "IBGPI_wireguard_tokyo"
}
```

字段说明：

- `ok`
  - 命令最终是否成功
- `applied`
  - 是否已经写入文件
- `output`
  - 节点执行输出
- `files`
  - 本次落盘文件
- `config_name`
  - 方便后端和日志对齐

## 节点执行行为

建议节点执行行为继续沿用现有 deploy 风格。

## 1. 基本流程

1. trim 所有字符串
2. 校验 payload
3. 计算输出路径：
   - `<bird_internal_dir>/<config_name>.conf`
4. 验证路径仍在目标目录内
5. 原子写入配置文件
6. 执行 `birdc configure`

本阶段不修改：

- `WireGuard` 配置文件
- `OSPF` 配置文件

## 2. ibgp.internal.remove

请求体：

```json
{
  "request_id": "6f67e8c1-8241-4d1a-a0da-2f4f9b79f3e9",
  "config_name": "IBGPI_wireguard_tokyo"
}
```

行为：

1. 校验 `request_id` 和 `config_name`
2. 解析内部 `iBGP` 文件路径
3. 验证路径安全
4. 删除文件
5. reload / configure `BIRD`

## 3. ibgp.internal.status

请求体：

```json
{
  "config_name": "IBGPI_wireguard_tokyo"
}
```

返回体：

```json
{
  "ok": true,
  "output": "bird protocol bgp detail"
}
```

作用：

- 后端可独立查询内部 `iBGP` 协议状态
- 不与 `WireGuard` 或 `OSPF` 状态混用

## 后端部署适配器

后端建议新增：

- `InternalIbgpPeerSpec`
- `InternalIbgpNodeSpec`
- `InternalIbgpDeployPayload`
- `build_internal_ibgp_deploy_payload()`
- `deploy_internal_ibgp_node()`

### 适配器职责

这个适配器只做：

- 从 `RoutingSession(kind="ibgp")` 和当前承载视图派生 deploy payload

它不直接负责建立承载或恢复 IGP。

## 后端状态回写

这次状态回写建议以：

- `RoutingSession`

为核心。

## 1. 成功回写

当节点 `ibgp.internal.deploy` 成功时：

- 把对应 `RoutingSession.oper_state` 更新为 `active`
- 写一条 deploy 结果日志
- 可选地记录一条 `FabricStateSnapshot`

## 2. 失败回写

当节点 `ibgp.internal.deploy` 失败时：

- 把对应 `RoutingSession.oper_state` 更新为 `failed`
- 记录节点返回的 `output`
- 不修改 `TransportLink.oper_state`
- 不修改 `RoutingSession(kind="ospf").oper_state`

## 3. remove 回写

当节点 `ibgp.internal.remove` 成功时：

- 把对应 `RoutingSession.oper_state` 更新为 `planned`

## 错误类型

建议后端与节点统一关心 4 类错误。

## 1. payload 结构错误

例如：

- 缺少 `config_name`
- `bird_config` 为空

节点应直接返回：

- `ok = false`
- `applied = false`

## 2. spec 渲染错误

例如：

- 缺少 `router_id`
- 缺少 `internal_asn`
- 缺少 `peer_address`

后端应在发起节点请求前直接失败。

## 3. 落盘错误

例如：

- 目录不存在
- 权限不足

节点应返回：

- `ok = false`
- `applied = false`

## 4. reload / configure 错误

例如：

- `birdc configure` 失败

节点应返回：

- `ok = false`
- `applied = true`

## 与现有系统的关系

## 1. 不替代 OSPF deploy

`ibgp.internal.deploy` 和 `ospf.internal.deploy` 并存。

职责分工：

- `internal.deploy`
  - 内部 `WireGuard` 传输层
- `ospf.internal.deploy`
  - 内部 IGP
- `ibgp.internal.deploy`
  - 内部 BGP 控制平面

## 2. 共用内部 BIRD 目录，但不共用协议命令

内部 `OSPF` 和内部 `iBGP` 可以共用：

- `bird_internal_dir`

但必须通过：

- 独立 `config_name`
- 独立命令空间
- 独立状态回写

保持协议层边界。

## 3. 不绑定 UI

本阶段后端可以先由内部 service 触发 deploy，不要求 portal 或 Telegram 立刻暴露入口。

## 测试策略

这一阶段后续实现时至少应补 4 类测试。

## 1. spec 构建测试

验证：

- `RoutingSession(kind="ibgp") -> InternalIbgpNodeSpec`
- 只纳入支持 `iBGP` 的节点和合法链路
- `WireGuard` 承载地址与 ASN 能正确进入 spec

## 2. payload 构建与服务测试

验证：

- `InternalIbgpNodeSpec -> InternalIbgpDeployPayload`
- 后端 `deploy/remove/status`
- 成功和失败时 `RoutingSession.oper_state` 回写

## 3. 节点执行测试

验证：

- 请求体校验
- 写文件成功
- `birdc configure` 成功
- `birdc configure` 失败

## 4. API 命令分发测试

验证：

- `ibgp.internal.deploy`
- `ibgp.internal.remove`
- `ibgp.internal.status`

在节点 API 层都能正确路由。

## 非目标

本阶段明确不做：

- `ZeroTier` 承载上的 `iBGP`
- route-reflector
- 复杂 BGP policy 编排
- 多协议联合回滚
- 自动拓扑推导

## 建议实现顺序

下一阶段建议按下面顺序落地：

1. `WireGuard iBGP spec builder`
2. `InternalIbgpDeployPayload` 与后端 deploy service
3. 节点 `ibgp.internal.deploy` / `ibgp.internal.remove` / `ibgp.internal.status`
4. `RoutingSession.oper_state` 回写
5. 聚焦回归测试

## 总结

这份设计的核心是：

- 不把 `iBGP` 塞回 `OSPF deploy`
- 不让 `iBGP` 重新承担承载或 IGP 职责
- 让它作为独立协议层接到现有内部网络栈上

当前第一版主路径应当是：

`InternalFabricRenderContext + RoutingSession(kind="ibgp") -> InternalIbgpNodeSpec -> InternalIbgpDeployPayload -> ibgp.internal.deploy -> 节点执行 -> RoutingSession.oper_state 回写`

这样后续无论继续接：

- `ZeroTier` 承载
- route-reflector
- 总编排器

都能在一条清晰、分层、可扩展的内部控制平面 deploy 链路上继续推进。
