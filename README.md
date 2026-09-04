# DevBox · 开发者工具箱

我参照IT-tools，写了一个IT工具网站,网站地址:https://devbox.1206616680.workers.dev/
对标 [it-tools.tech](https://it-tools.tech/) 的精简版，只保留 **24 个高频工具**，砍掉了 it-tools 里那些几乎没人点的功能（ASCII 艺术、emoji 选择器、假数据生成、扭矩/温度单位换算、Docker Compose 转换、chmod 计算器、IBAN/BIC 校验、宝可梦 IV 计算器等）。

所有计算都在浏览器本地完成，**没有任何后端、不上传任何数据**，纯静态站点。

## 收录的工具

| 分类 | 工具 |
| --- | --- |
| 转换 | JSON 格式化/校验、JSON ⇄ YAML、时间戳转换、进制转换、Cron 表达式、SQL 格式化 |
| 编码 | Base64 编解码、文件转 Base64、URL 编解码、HTML 实体、JWT 解析、Unicode/转义 |
| 加密 | 哈希计算（MD5/SHA 系列）、HMAC 签名、AES 加解密 |
| 生成器 | UUID v4/v7 + NanoID、密码生成器、二维码生成 |
| 网络 | IPv4 子网计算、URL 解析 |
| 文本 | 正则测试、文本对比 Diff、文本处理与统计 |
| 图像 | 颜色转换 |

### 「使用权重」是怎么起作用的

1. **收录层面**：工具清单按社区使用频率筛选，每个工具在 `src/lib/registry.ts` 里带一个 `base` 初始热度。
2. **个性化层面**：`src/lib/usage.ts` 在 localStorage 记录你每次打开工具的次数和时间，权重 = `使用次数 + 时间衰减加成`（半衰期约 10 天）。首页「常用」区块按这个权重实时重排，用得多的自动浮到最前。
3. 还可以点右上角 ★ 收藏，收藏的工具固定显示在首页顶部、侧栏加星标。

数据只存在浏览器本地，换设备不同步。

## 本地开发

```bash
npm install
npm run dev
```

## 部署到 Cloudflare

用的是 **Workers 静态资源**（Static Assets），比 Pages 更适合绑自定义域名。

```bash
npx wrangler login
npm run deploy
```

`npm run deploy` 会先 `vite build` 再 `wrangler deploy`，部署完会给你一个 `devbox.<你的子域>.workers.dev` 地址。

### 绑定自己的域名

域名需要已经托管在同一个 Cloudflare 账号下。在 `wrangler.jsonc` 里加上 `routes`：

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

改完再跑一次 `npm run deploy`，Cloudflare 会自动创建 DNS 记录和证书，一两分钟后就能访问。

> 想改 Worker 名字（影响 workers.dev 子域）就改 `wrangler.jsonc` 的 `name` 字段。

## 增加一个新工具

1. 在 `src/tools/` 新建组件，默认导出一个无 props 的 React 组件；
2. 在 `src/lib/registry.ts` 的 `TOOLS` 数组里加一行 `t(id, 名称, 描述, 分类, 初始热度, 关键词, () => import('../tools/Xxx'))`。

路由、侧栏、搜索、权重排序会自动接上，不需要改别的地方。`src/components/ui.tsx` 里有现成的 `Card / Field / Input / Textarea / Select / Toggle / Result / Row / Badge / CopyButton` 可以直接用。

## 技术栈

React 18 + TypeScript + Vite 5 + Tailwind CSS 4，每个工具单独 code-split，首屏只有 ~90KB gzip。
