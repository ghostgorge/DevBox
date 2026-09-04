---
title: "DevBox：把 it-tools 砍掉一半，做一个只留高频功能的开发者工具箱"
excerpt: "it-tools 有 80 多个工具，但我常用的不到 20 个。于是我按使用频率重做了一个：24 个工具、纯前端、零后端，一条命令部署到 Cloudflare。"
categories: ["开发工具"]
tags: ["it-tools", "开发者工具", "Cloudflare Workers", "React", "Vite", "Tailwind CSS", "开源"]
---

## 起因：好用，但太胖了

[it-tools.tech](https://it-tools.tech/) 大概是这几年最好用的在线开发工具箱之一 —— JSON 格式化、Base64、JWT 解析、哈希、UUID，随手打开就能用，不用装客户端。

但用久了会发现一个问题：它有 **80 多个工具**，而我真正会反复打开的不到 20 个。剩下的那些 —— ASCII 艺术生成、emoji 选择器、扭矩单位换算、IBAN 校验、chmod 计算器、甚至宝可梦 IV 计算器 —— 常年躺在侧边栏里，唯一的作用是让我找东西时多滚两屏。

所以我做了 **DevBox**：同样的思路，但**只保留高频工具**，并且让它**按你自己的使用习惯自动重排**。

## 收录了什么

一共 24 个，分成 7 类：

| 分类 | 工具 |
| --- | --- |
| 转换 | JSON 格式化 / 校验、JSON ⇄ YAML、时间戳转换、进制转换、Cron 表达式、SQL 格式化 |
| 编码 | Base64 编解码、文件转 Base64、URL 编解码、HTML 实体、JWT 解析、Unicode / 转义 |
| 加密 | 哈希计算（MD5 / SHA-1 / SHA-256 / SHA-512 / SHA3 / RIPEMD160）、HMAC 签名、AES 加解密 |
| 生成器 | UUID v4 / v7 + NanoID、密码生成器、二维码生成 |
| 网络 | IPv4 子网计算、URL 解析 |
| 文本 | 正则测试、文本对比 Diff、文本处理与统计 |
| 图像 | 颜色转换 |

几个我自己用得最多、也做得最细的：

- **JSON 格式化**：除了美化 / 压缩 / 校验，还能按 key 排序，以及用 `data.items[0].name` 这样的路径直接把想要的字段抠出来。
- **JWT 解析**：Header / Payload 分开展示，`exp` `iat` `nbf` 自动转成人类可读时间，并直接标出这个 token「已过期」还是「有效期内」。
- **正则测试**：匹配结果在原文里高亮，编号分组和命名分组分别列出，下方还有一个替换预览框，`$1` `$<name>` 实时生效。
- **Cron 表达式**：五段分别标注含义，中文描述（"在 09:00, 星期一至星期五"），并列出接下来 6 次实际执行时间。
- **哈希计算**：七种算法同时算，支持直接拖文件进来算文件摘要，Hex / Base64 双格式。
- **IPv4 子网计算**：给一个 `192.168.1.100/24`，把网络地址、广播地址、可用主机范围、反掩码、地址类别、二进制掩码全列出来。

## 最有意思的部分：让「使用权重」真正起作用

标题里说「按使用频率」，这件事在 DevBox 里落在两个层面。

### 第一层：决定收录谁

工具清单本身就是按社区使用频率筛的。注册表里每个工具带一个初始热度分：

```ts
t('json-format', 'JSON 格式化 / 校验', '美化、压缩、校验 JSON，并可提取 JSONPath', '转换', 100, [...])
t('hash',        '哈希计算',           'MD5 / SHA-1 / SHA-256 / SHA-512 等摘要',    '加密',  90, [...])
t('color',       '颜色转换',           'HEX / RGB / HSL 互转与取色板',              '图像',  54, [...])
```

新用户第一次打开，首页「热门工具」区块就是按这个分数排的 —— 不至于一片空白，也不至于把冷门工具摆在第一屏。

### 第二层：按你自己的习惯重排

每次打开一个工具，浏览器本地会记一笔次数和时间戳。权重的算法是**使用次数 + 时间衰减加成**：

```ts
export function weightOf(id: string, usage: UsageMap): number {
  const rec = usage[id]
  if (!rec) return 0
  const days = (Date.now() - rec.last) / 86_400_000
  const recency = Math.exp(-days / 14)   // 半衰期约 10 天
  return rec.n + recency * 3
}
```

为什么要加时间衰减，而不是单纯按次数排？因为「上个月密集用了三天」和「这周天天在用」是两回事。指数衰减让最近的行为有额外加成，但又不会让一次偶然的点击就把某个工具顶到第一 —— 累计次数仍然是主项。

用几天之后，首页第一屏基本就是你的个人工具集了。

### 一个刻意的取舍：侧栏不跟着动

我一开始让左侧导航栏也按权重排序，结果非常难用 —— 每次点开一个工具，整个侧栏就重新洗牌一次，肌肉记忆完全失效。

所以最终版本是：**首页按权重排，侧栏按分类固定排**。只有在搜索框里输入内容时，侧栏才切换成按相关度 + 权重排序的扁平结果列表。

另外还有一层手动兜底：每个工具右上角有 ★，收藏后固定显示在首页顶部，侧栏里加星标。算法猜错了的时候，人可以直接改。

## 隐私：没有后端，所以没什么好承诺的

DevBox 是**纯静态站点，没有任何服务端逻辑**。所有计算 —— 哈希、AES 加解密、JWT 解析、文件转 Base64 —— 全部在你的浏览器里跑完。

这不是一句「我们承诺不会记录您的数据」的隐私政策，而是架构上就没有可以记录数据的地方。你可以打开开发者工具的 Network 面板确认：除了首次加载的静态资源，一个请求都没有。

顺带的好处：**断网也能用**。页面加载完之后拔网线，所有工具照常工作。

使用记录和收藏存在 `localStorage`，只在这台设备的这个浏览器里，不会同步、也不会上传。

## 技术栈与体积

React 18 + TypeScript + Vite 5 + Tailwind CSS 4。

每个工具都是独立的懒加载 chunk（共 29 个 JS 分片），首屏只加载框架和外壳：

| 资源 | gzip 后 |
| --- | --- |
| vendor（React + Router） | 53.6 KB |
| 应用外壳 | 26.1 KB |
| CSS | 5.8 KB |
| **首屏合计** | **约 86 KB** |

打开某个具体工具时才拉它自己的那一片，绝大多数工具单片在 1–4 KB。少数依赖较重的（Cron 用了 cron-parser + cronstrue 约 58 KB，SQL 格式化约 77 KB）因为是懒加载，不用就不下载。

## 部署：一条命令上 Cloudflare

用的是 **Cloudflare Workers Static Assets**（不是 Pages），主要因为绑自定义域名更直接。

```bash
npm install
npx wrangler login
npm run deploy
```

`npm run deploy` 会先 `vite build` 再 `wrangler deploy`，结束后给你一个 `devbox.<你的子域>.workers.dev` 地址。

绑自己的域名（域名需已托管在同一个 Cloudflare 账号下），在 `wrangler.jsonc` 里加一段：

```jsonc
{
  "name": "devbox",
  "compatibility_date": "2025-01-01",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  },
  "routes": [
    { "pattern": "tools.yourdomain.com", "custom_domain": true }
  ]
}
```

再跑一次 `npm run deploy`，Cloudflare 自动创建 DNS 记录和证书，一两分钟生效。

其中 `not_found_handling: "single-page-application"` 是关键 —— 有了它，`/t/json-format` 这样的深链接直接访问或刷新才不会 404。

## 加一个新工具要写多少代码

两步：

1. 在 `src/tools/` 新建一个组件，默认导出一个无 props 的 React 组件；
2. 在 `src/lib/registry.ts` 的数组里加一行：

```ts
t('my-tool', '工具名', '一句话描述', '文本', 50,
  ['keyword', 'guanjianci'], () => import('../tools/MyTool')),
```

路由、侧栏、搜索（支持中文名 / 描述 / 英文关键词 / 拼音）、权重排序、收藏，全部自动接上，不需要改任何其他文件。

`src/components/ui.tsx` 里有一套现成的组件 —— `Card` `Field` `Input` `Textarea` `Select` `Toggle` `Result` `Row` `Badge` `CopyButton` `DownloadButton` —— 其中 `Result` 自带复制和下载按钮，一个典型工具的实现通常在 60–100 行之内。

## 已知的取舍

写在最后，免得像广告：

- **`crypto-js` 已停止维护**。哈希和 AES 用的是它，功能没问题，但如果在意长期维护性，可以改用浏览器原生 WebCrypto —— 代价是 MD5 得自己实现（WebCrypto 不提供 MD5）。
- **Cron 和 SQL 格式化的依赖偏重**。懒加载抵消了首屏影响，但真要抠体积，这两个可以换更轻的实现。
- **不做跨设备同步**。使用记录只在本地，这是「没有后端」的直接代价 —— 我认为这个交换是划算的。
- **AES 工具的密钥处理是简化版**（不足 16 位补零），适合调试和临时解密，不要拿它当生产环境的加密方案。

## 小结

it-tools 的价值在于「什么都有」，DevBox 的价值在于「打开就是你要的那个」。

如果你也觉得每天用的工具就那么十几个，剩下的只是噪音，不妨自己 fork 一份，把 `registry.ts` 里的清单改成你自己的 —— 那才是这个项目真正的用法。
