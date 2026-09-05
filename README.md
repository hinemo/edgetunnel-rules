# edgetunnel-rules

EdgeTunnel 2.1 节点 + 本仓库分流规则 → 统一 Mihomo/Nikki 订阅 的**规则端**。

- EdgeTunnel 2.1 负责：生成 VLESS + XHTTP + Cloudflare 节点
- 本仓库负责：分流规则、业务组定义、国家识别、测速参数、三级代理组结构

## 仓库结构

```
config.yaml           规则元数据：业务组 / 规则文件 / 评估顺序 / 测速 / 国家识别
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
