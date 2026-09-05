# edgetunnel-rules

EdgeTunnel 2.1 节点 + 本仓库分流规则 → 统一 Mihomo/Nikki 订阅 的**规则端**。

- EdgeTunnel 2.1 负责：生成 VLESS + XHTTP + Cloudflare 节点
- 本仓库负责：分流规则、业务组定义、国家识别、测速参数、三级代理组结构

## 仓库结构

```
config.yaml           规则元数据：业务组 / 规则文件 / 评估顺序 / 测速 / 国家识别
template/
  proxy-groups.yaml   三级代理组结构模板（业务组 → 模式 → 国家 → 手动HA）
rules/
  cn-direct.list      🇨🇳 国内直连
  academic-ai.list    🎓 学术/AI + 开发工具
  media.list          🌍 媒体/社交
  google.list         🌐 Google 服务
  apple.list          🍎 Apple 服务
```

## 规则文件格式

每行一条规则，`#` 开头为注释：

| 写法 | 含义 |
|---|---|
| `example.com` | DOMAIN-SUFFIX（含所有子域） |
| `full:example.com` | 仅精确匹配该域名 |
| `keyword:xxx` | 域名包含 xxx 即命中 |

## 规则评估顺序（config.yaml 中 rule_groups 从上到下）

1. 🇨🇳 国内 → DIRECT
2. 🎓 学术/AI（先于 Google，保证 `copilot.microsoft.com` 等精确规则不被误伤）
3. 🌍 媒体/社交
4. 🌐 Google/Apple
5. 🕸️ 漏网之鱼（`MATCH` 兜底，规则文件中不写）

> 注：`gemini.google.com` 会被 `google.com` 后缀规则命中，默认归 Google/Apple；
> 如需强制归 🎓 学术/AI，在 `academic-ai.list` 加一行 `full:gemini.google.com` 即可（该组已排在 Google 前）。

## 与 EdgeTunnel 2.1 合并

生成器 / 订阅转换模板读取 `config.yaml` + `rules/`，与 EdgeTunnel 2.1 输出的节点合并后生成统一订阅：

- 测速：`https://cp.cloudflare.com/generate_204`，`interval: 300`，`tolerance: 50`
- 国家池：按 `config.yaml` 中 `countries[].match` 正则对节点名分类
- 业务组模式：🚀 自动（全节点最快）/ 🌏 国家（国家内最快）/ 🎯 手动（指定节点 + 故障转移）
- 代理组结构：见 `template/proxy-groups.yaml`（三级结构模板，生成器套用后输出最终订阅）

## 测速规则语义（重要）

- 测的是**完整代理链路**：客户端 → VLESS → XHTTP → Cloudflare → 源站 → 测试 URL。
  **不是** ping Cloudflare IP（多个 CF 节点可能落在同一边缘网络，ping 会测偏）。
- 测试 URL 固定 `https://cp.cloudflare.com/generate_204`（Mihomo 官方示例同款）；
  **不用** `https://www.gstatic.com/generate_204`——避免把 Google 网络路径混入 CF+XHTTP 节点测速。
- 健康判定：**经代理拿到 HTTP 204/200 才算健康**；TCP 能连不算（防止“TCP 正常但 Google 打不开”）。
- `interval: 300`（秒）、`tolerance: 50`（ms）：避免 2ms 级抖动导致乒乓切换。

## Cloudflare / XHTTP 节点核对清单（接入 EdgeTunnel 2.1 前必查）

生成器必须**原样透传** EdgeTunnel 2.1 订阅里的 XHTTP 参数，不重新发明字段映射：

| 字段 | 核对点 |
|---|---|
| `network` | 必须为 `xhttp`，且目标 Nikki/Mihomo 版本支持 xhttp transport |
| `xhttp-opts.mode` | 与源站一致：`stream-one` / `packet-up` / `stream-up` |
| `xhttp-opts.path` | 订阅输出实际 path |
| `xhttp-opts.host` | 订阅输出实际 host |
| `xhttp-opts.extra` | 订阅带 extra 则原样保留 |
| `encryption` | 新版 Xray VLESS+XHTTP 有 Encryption 迁移，按源站确认 |
| `xmux` | 源站启用 XMUX 时原样映射 |

节点命名必须含国家标识（如 `🇯🇵 日本 01`、`JP01`），否则 `countries[].match` 正则分不出国家池。
