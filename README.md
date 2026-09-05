# edgetunnel-rules

EdgeTunnel 2.1 节点 + 本仓库分流规则 → 统一 Mihomo/Nikki 订阅 的**规则端**。

- EdgeTunnel 2.1 负责：生成 VLESS + XHTTP + Cloudflare 节点
- 本仓库负责：分流规则、业务组定义、国家识别、测速参数、三级代理组结构

## 订阅转换配置地址（EdgeTunnel 2.1 后台填这个）

```
https://raw.githubusercontent.com/hinemo/edgetunnel-rules/main/subconverter/config.ini
```

- EdgeTunnel 2.1 后台 → **订阅转换配置** → **配置文件地址（SUBCONFIG）** 填上面的 URL
- EdgeTunnel 会把生成的节点交给 subconverter，用这份 ini（业务组 + 三级模式 + CF 测速 + 本仓库规则）合并出**带规则的订阅**
- 最终 VPN 订阅地址 = 你自己的 EdgeTunnel 订阅地址（`https://<你的域名>/sub?token=<TOKEN>`，或快速路径 `/<KEY>`）

## 仓库结构

```
config.yaml           规则元数据：业务组 / geosite 类目 / 补充规则 / 评估顺序 / 测速 / 国家识别
template/
  proxy-groups.yaml   三级代理组结构模板（业务组 → 模式 → 国家 → 手动HA）
rule-sets/
  cn.list             🇨🇳 国内直连补充（Clash 通用格式）
  media.list          🌍 媒体/社交（geosite 类目解析后的通用域名列表）
  google.list         🌐 Google/Apple
  academic.list       🎓 学术/AI
subconverter/
  config.ini          EdgeTunnel 2.1 订阅转换配置文件（SUBCONFIG 地址）
rules/
  cn-direct.list      🇨🇳 国内直连补充（geosite:cn 之外的常用站点）
  academic-ai.list    🎓 学术/AI 补充（geosite 无类目的 AI/开发工具）
  media.list          🌍 媒体/社交补充（geosite 无类目的服务，如 Snapchat）
```

## 规则形式：GEOSITE 为主，.list 只做补充

大型服务用 Mihomo 原生 `GEOSITE` 类目，**一行顶几百行**，随设备端 geosite.dat 自动更新（Nikki 已开启 `geox_auto_update`）：

- 🌐 Google/Apple → `GEOSITE,google` + `GEOSITE,apple`（不再维护 google.list / apple.list）
- 🌍 媒体/社交 → `GEOSITE,youtube`、`netflix`、`disney`、`spotify`、`telegram`、`twitter`、`x`、`facebook`、`instagram`、`tiktok`、`twitch`、`reddit`、`discord`、`pinterest`、`linkedin`、`whatsapp`、`line`、`vimeo`、`tumblr`、`dailymotion`、`medium`、`quora`
- 🎓 学术/AI → `GEOSITE,openai` + `anthropic` + `github`
- 🇨🇳 国内 → `GEOSITE,cn` + `GEOIP,CN`（生成器负责）

`.list` 只保留 geosite 没有类目的：Snapchat（media）、Copilot/Gemini/其他 AI 站（academic）、常用国内站点（cn）。

> 为什么不用 `DOMAIN-KEYWORD,google` 这类通配？它会误伤 `google.cn` / `apple.com.cn`（这些必须直连）；Mihomo 没有 `*.google.*` 这类 glob，`GEOSITE` 才是官方维护的“通配集合”。

## 规则文件格式（.list 补充文件）

每行一条规则，`#` 开头为注释：

| 写法 | 含义 |
|---|---|
| `example.com` | DOMAIN-SUFFIX（含所有子域） |
| `full:example.com` | 仅精确匹配该域名 |
| `keyword:xxx` | 域名包含 xxx 即命中 |

## 跨客户端兼容（rule-sets）

`rule-sets/*.list` 是把 geosite 类目解析后的**通用域名列表**（`DOMAIN-SUFFIX,xxx` 格式），
供没有内置 geosite 的客户端使用，也方便 Clash 系做 rule-provider：

| 客户端 | 用法 |
|---|---|
| Mihomo / Clash.Meta / Clash Verge | `GEOSITE,google` 直写；或把 `rule-sets/*.list` 转 rule-provider |
| sing-box | `geosite:google` 内置；或 `rule_set` 引用 `rule-sets/*.list` |
| Surge / Stash / Shadowrocket | `RULE-SET,https://raw.githubusercontent.com/hinemo/edgetunnel-rules/main/rule-sets/media.list,🌍 媒体/社交` |
| Quantumult X | `filter_local` / 远程 filter 引用 `rule-sets/*.list` 转 host-suffix |

raw 地址示例：`https://raw.githubusercontent.com/hinemo/edgetunnel-rules/main/rule-sets/google.list`

## 规则评估顺序（config.yaml 中 rule_groups 从上到下）

1. 🇨🇳 国内 → DIRECT（geosite:cn + cn-direct.list）
2. 🎓 学术/AI（先于 Google，保证 `gemini.google.com` / `copilot.microsoft.com` 归本组）
3. 🌍 媒体/社交
4. 🌐 Google/Apple
5. 🕸️ 漏网之鱼（`MATCH` 兜底，规则文件中不写）

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
