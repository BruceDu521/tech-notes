# 在 macOS 上让 Clash Party 与 Tailscale 共存

> 从路由冲突、MagicDNS 和 DERP 排障，到使用 Mihomo 原生 Tailscale outbound 的可复现方案。

## 背景

2026 年 7 月，我在 macOS 上同时运行：

- Clash Party，Mihomo Core `v1.19.27`
- Clash TUN 模式
- Tailscale Standalone `v1.98.8`
- MongoDB Compass、浏览器和本地 Electron 开发应用

此前这套环境可以正常访问 tailnet 内的数据库和管理站点，但在没有主动修改配置的情况下，访问突然变得极慢，最终接近完全不可用。应用层看到的症状包括：

- 本地后端请求等待数分钟后超时；
- MongoDB driver 的 server monitor 和连接池不断超时；
- Compass 能显示“连接中”，但 schema 和 collection 长时间无法加载；
- tailnet 内的管理站点也无法打开。

这不是单一应用的 bug，而是 Clash TUN、系统 Tailscale、MagicDNS 和当前网络路径共同造成的问题。

## 为什么“什么都没改”也可能突然出问题

配置文件没有变化，不代表运行时网络状态没有变化。以下状态都可能动态改变：

- NAT 映射和 UDP 可达性；
- Tailscale peer-to-peer 与 DERP relay 的选择；
- Wi-Fi、睡眠唤醒或 Network Extension 重建后的路由优先级；
- Clash TUN 与 Tailscale `utun` 接口的创建顺序；
- 代理节点、DNS 缓存和连接复用状态。

因此，同一份配置可能上午可用，下午就因流量从直连退化到 DERP，或因 TUN 抢走 tailnet 路由而变得不可用。

## 先确定问题在哪一层

不要一开始就反复重启业务应用。推荐按层排查：

1. **应用层**：确认超时发生在 HTTP、数据库握手还是读取数据阶段。
2. **DNS 层**：确认 tailnet FQDN 是否由正确的 resolver 处理。
3. **路由层**：检查 `100.64.0.0/10` 和 Tailscale IPv6 ULA 的实际出口。
4. **Tailscale 层**：用 `tailscale netcheck` 和 `tailscale ping` 判断 UDP、DERP 和 direct 状态。
5. **传输层**：对目标端口做 TCP/TLS 握手测试，区分“能解析”和“能传输”。

这次排障中，关键证据是：

- 部分 peer 始终经 DERP，无法升级为 direct；
- Clash TUN 开启时，tailnet 域名被普通代理节点接管；
- 即使暂时恢复 TLS 握手，长连接和 server monitor 仍会再次超时；
- 问题同时影响数据库和管理站点，因此不是 MongoDB 自身故障。

## 尝试过但不够稳定的方案

### 1. 在 Clash TUN 中排除 CGNAT 网段

在 Clash Party 的 TUN 设置中加入：

```text
100.64.0.0/10
```

这能让系统 Tailscale 重新接管 tailnet IPv4 路由，适合作为快速验证手段。但它不能解决所有问题：如果系统 Tailscale 到目标 peer 仍长期走质量不佳的 DERP，数据库连接依然会抖动或超时。

另外，Clash Party 的公共配置、订阅覆写和最终生成配置是不同层级。必须检查最终生成的 `route-exclude-address`，不能只看 UI 或某个覆写文件。

### 2. 仅添加 `IP-CIDR,100.64.0.0/10`

在 fake-IP 模式下，应用连接常以域名元数据和 `198.18.0.0/16` fake IP 进入 Mihomo。只有 `IP-CIDR` 规则时，请求可能根本不会匹配 tailnet 地址，而是落入最后的 `MATCH` 规则并走普通代理。

日志中的典型表现是：

```text
match Match using <fallback proxy group>
```

因此必须为实际使用的 tailnet FQDN 和私有域名添加 `DOMAIN-SUFFIX` / `DOMAIN` 规则，而且要放在 catch-all 规则之前。

### 3. 给原生 Tailscale outbound 配置 `dialer-proxy`

Mihomo 的 Tailscale outbound 支持用 `dialer-proxy` 把 control plane、DERP 和 STUN 流量送进另一个代理。理论上很适合代理网络，但在此次 macOS + Clash Party 组合中，无论指向策略组还是明确声明支持 UDP 的具体节点，都会失败：

```text
tsnet: wgengine: magicsock: Rebind IPv4 failed:
failed to bind any ports (tried [0])
```

这表示规则已经命中 Tailscale outbound，但 tsnet 在登录前就无法通过该代理创建 MagicSock UDP socket。因此不会产生交互式登录 URL。

最终的解决办法是：**完全删除 `dialer-proxy` 字段**，而不是把它改成 `DIRECT`。

## 最终可用方案：Mihomo 原生 Tailscale outbound

Mihomo `v1.19.25` 起支持基于 tsnet 的原生 Tailscale outbound。它作为一个普通 outbound 节点，只处理规则明确交给它的流量，不需要让系统 Tailscale 与 Clash 同时争夺整机路由。

下面是脱敏后的 Clash Party 覆写示例：

```yaml
proxies+:
  - name: tailscale
    type: tailscale
    hostname: macbook-mihomo
    state-dir: tailscale
    ephemeral: false
    # 这次访问 MongoDB 和 HTTPS 管理站点只需要 TCP。
    # udp: false 同样可以正常工作；它不是 Rebind 错误的根因。
    udp: false
    accept-routes: true
    ip-version: ipv4-prefer

+rules:
  # 用你自己的 tailnet DNS suffix 替换 example-tailnet.ts.net
  - DOMAIN-SUFFIX,example-tailnet.ts.net,tailscale

  # 只添加确实通过 tailnet 访问的私有域名，不要笼统接管所有公司域名
  - DOMAIN,private-admin.example.com,tailscale

  - IP-CIDR,100.64.0.0/10,tailscale,no-resolve
  - IP-CIDR6,fd7a:115c:a1e0::/48,tailscale,no-resolve

# 某些自定义域名虽然指向 tailnet IP，但 Tailscale outbound 按域名拨号
# 仍可能超时。固定到实际 tailnet IP 后，Mihomo 会直接把 IP 交给
# Tailscale outbound。请替换为自己的地址。
hosts:
  private-admin.example.com: 100.80.81.61
```

注意：这个可用配置里**没有** `dialer-proxy`。

如果环境需要 Tailscale 自己处理 MagicDNS / split DNS，可以使用 Mihomo
为该 outbound 注册的原生 resolver：

```yaml
dns:
  respect-rules: true
  nameserver-policy:
    "+.example-tailnet.ts.net": "tailscale://tailscale"

  # tailnet 域名必须返回真实 100.x 地址，不能继续分配 198.18.x.x fake IP。
  fake-ip-filter:
    - "+.example-tailnet.ts.net"
```

`nameserver-policy` 和 `fake-ip-filter` 缺一不可：前者选择 Tailscale DNS，
后者确保应用拿到真实 tailnet IP，而不是 Mihomo 的 `198.18.0.0/16`
fake IP。

### Clash Party 的 DNS 覆写优先级陷阱

Clash Party 的 YAML 覆写优先级低于应用级“DNS 覆写”。如果主界面的
“DNS 覆写”已开启，即使 YAML 中写了：

```yaml
dns:
  fake-ip-filter+:
    - "+.example-tailnet.ts.net"
```

最终生成配置里也可能没有这一项。此次排障中的实际表现是：

- 自定义管理域名通过 `hosts` 后已经连接成功；
- 页面依赖的 MagicDNS API 主机仍显示为 `198.18.x.x`；
- 日志持续出现 `match DomainSuffix/... using tailscale`，随后
  `context deadline exceeded`；
- MongoDB 的其他 tailnet 连接同时可以成功，说明 Tailscale 数据面正常。

处理方式是在 Clash Party 主界面点击“DNS 覆写”卡片，将
`+.example-tailnet.ts.net` 加入那里的 `fake-ip-filter` 列表，再重启 Core。
随后检查最终生成配置，而不是只看 YAML 覆写编辑器。

### 用 IP + TLS 测试区分传输与解析问题

如果域名访问超时，可以让 HTTPS 代理直接连接目标 tailnet IP，同时保留
原域名作为 SNI：

```bash
openssl s_client \
  -proxy 127.0.0.1:7890 \
  -connect 100.80.81.61:443 \
  -servername private-admin.example.com \
  -brief </dev/null
```

若该测试成功、证书校验也正确，但域名方式仍超时，则已经排除目标服务、
ACL、TCP 443 和 TLS。问题应继续沿域名解析和 fake-IP 元数据路径排查，
不应再反复调整 UDP 或 DERP。

## 首次登录步骤

1. Clash 使用 **Rule** 模式并开启 TUN。
2. 如果改用 Mihomo 原生 Tailscale outbound，清除之前为系统 Tailscale 添加的 `100.64.0.0/10` TUN 排除项。
3. 断开系统 Tailscale GUI，避免同时维护两套 tailnet 数据面。
4. 保存覆写并重启 Mihomo Core。
5. 发起一次能命中 `DOMAIN-SUFFIX` 或 `DOMAIN` 规则的请求。
6. 在 Clash Party 实时日志中搜索 `tailscale`。
7. 打开首次启动输出的交互式登录 URL，在浏览器中完成授权。
8. 重新发起请求。首次连接可能因为 lazy initialization 超时，正常情况下重试即可。

登录 URL、auth key 和 state directory 都属于敏感信息，不应粘贴到聊天、issue、日志收集系统或公开仓库。

## 日志快速判断表

| 日志或现象 | 含义 | 处理方式 |
| --- | --- | --- |
| 搜索 `tailscale` 完全没有新日志 | 请求没有命中 Tailscale outbound | 检查 `DOMAIN-SUFFIX` / `DOMAIN` 顺序和最终生成配置 |
| `match Match using <fallback>` | 请求落入 catch-all 普通代理 | 增加并前置域名规则 |
| `match DomainSuffix/...` 后出现 `Rebind IPv4 failed` | 已命中 outbound，但 MagicSock 无法绑定 | 删除 `dialer-proxy`，重启 Core |
| 出现交互式登录 URL | tsnet 已成功启动，等待授权 | 本地打开 URL，切勿分享 |
| 第一次请求超时，第二次成功 | outbound lazy initialization | 属于预期行为，可重试 |
| 直接连接 tailnet IP 的 TCP/TLS 成功，但同一服务域名超时 | 数据面和服务正常，域名解析路径异常 | 使用 `hosts` 固定自定义域名，或配置 Tailscale DNS |
| 管理域名成功，但下游 `*.ts.net` API 仍显示 `198.18.x.x` 并超时 | 应用级 DNS 覆写覆盖了 YAML 的 fake-IP filter | 在 Clash Party“DNS 覆写”中加入 tailnet suffix，并检查最终配置 |
| 能登录但长期只走慢速 relay | 运行时路径质量问题 | 检查 DERP、UDP、NAT，必要时使用更靠近服务端的 relay/peer |

## 安全和维护建议

- 优先使用交互式登录，不把 `auth-key` 写进公开配置。
- `state-dir` 应持久化并限制文件权限；不要提交其中内容。
- 规则只覆盖明确属于 tailnet 的域名，避免将所有公司公网域名都送入 Tailscale。
- 修改后始终检查 Clash Party 的**最终生成配置**，确认覆写真正生效。
- 公开排障笔记不要包含真实 tailnet suffix、内部主机名、数据库 URI、邮箱、代理订阅或登录截图。
- 如果原生 outbound 在当前平台仍无法工作，可退回系统 Tailscale + TUN route exclusion，或使用 userspace networking + SOCKS5 的方案。

## 参考资料

- [Mihomo Tailscale outbound 文档](https://wiki.metacubex.one/en/config/proxies/tailscale/)
- [Mihomo hosts 文档](https://wiki.metacubex.one/en/config/dns/hosts/)
- [Mihomo DNS 配置文档](https://wiki.metacubex.one/en/config/dns/)
- [Mihomo PR #2786：Tailscale outbound support](https://github.com/MetaCubeX/mihomo/pull/2786)
- [Clash Party YAML 覆写优先级与合并规则](https://clashparty.org/docs/guide/override/yaml)
- [Tailscale 与 Clash 共存的一种 userspace SOCKS5 方案](https://jiz4oh.com/2024/09/tailscale-with-clash/)
