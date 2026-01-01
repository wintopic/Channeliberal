# ChannelPro 开发文档

> 将 Telegram Channel 转换为 MicroBlog 的 Web 应用

## 项目概述

**核心功能**: 从 Telegram 公开频道抓取内容，以博客形式展示，支持 SEO、RSS、零客户端 JS。

**技术栈**: Astro 5.x (SSR) + Cheerio + ofetch + dayjs + PrismJS + LRU Cache

## 目录结构

```
src/
├── pages/           # 路由页面
├── components/      # Astro 组件 (header, item, list)
├── layouts/         # 页面布局 (base.astro)
├── lib/             # 核心库
│   ├── telegram/index.js  # ⭐ 核心: Telegram 数据抓取
│   ├── env.js             # 环境变量读取
│   ├── dayjs.js           # 日期格式化
│   └── prism.js           # 代码高亮
└── assets/          # CSS 和图标
```

## 核心数据流

```
用户请求 → Astro 页面 → getChannelInfo() → LRU Cache
                                              ↓ miss
                                         Telegram API (t.me/s/{channel})
                                              ↓
                                         Cheerio 解析 HTML
                                              ↓
                                         返回 ChannelInfo
```

## 数据结构

### ChannelInfo
```javascript
{
  title: string,           // 频道名称
  description: string,     // 频道描述
  avatar: string,          // 头像 URL
  posts: Post[]            // 帖子数组
}
```

### Post
```javascript
{
  id: string,              // 帖子 ID
  title: string,           // 标题 (首行/首句)
  type: 'text' | 'service',
  datetime: string,        // ISO 时间
  tags: string[],          // 标签
  text: string,            // 纯文本
  content: string          // HTML 内容
}
```

## 路由表

| 路由 | 文件 | 说明 |
|------|------|------|
| `/` | `index.astro` | 首页 |
| `/posts/[id]` | `posts/[id].astro` | 单篇帖子 |
| `/before/[cursor]` | `before/[cursor].astro` | 向前翻页 |
| `/after/[cursor]` | `after/[cursor].astro` | 向后翻页 |
| `/search/[q]` | `search/[q].astro` | 搜索 |
| `/tags` | `tags.astro` | 标签云 |
| `/rss.xml` | `rss.xml.js` | RSS 订阅 |
| `/static/[...url]` | `static/[...url].js` | 静态资源代理 |

## 环境变量

| 变量 | 必填 | 说明 |
|------|------|------|
| `CHANNEL` | ✅ | Telegram 频道用户名 |
| `LOCALE` | | 语言 (zh-cn, en) |
| `TIMEZONE` | | 时区 (Asia/Shanghai) |
| `TELEGRAM/TWITTER/GITHUB` | | 社交链接 |
| `STATIC_PROXY` | | 静态资源代理 URL |
| `COMMENTS` | | 启用评论 |

## 核心函数

### `getChannelInfo(options)` - [src/lib/telegram/index.js](src/lib/telegram/index.js)

主入口函数，获取频道信息和帖子列表。

```javascript
const info = await getChannelInfo({
  channel: 'channelname',  // 频道名
  before: '123',           // 可选: 分页游标
  after: '456',            // 可选: 分页游标
  q: 'keyword'             // 可选: 搜索关键词
})
```

**缓存策略**: LRU Cache, 5分钟 TTL, 50MB 上限

## 二次开发指南

### 添加新页面
1. 在 `src/pages/` 创建 `.astro` 文件
2. 导入 `getChannelInfo` 获取数据
3. 使用 `BaseLayout` 包裹内容

```astro
---
import BaseLayout from '../layouts/base.astro'
import { getChannelInfo } from '../lib/telegram'
const info = await getChannelInfo({ channel: import.meta.env.CHANNEL })
---
<BaseLayout title="页面标题" info={info}>
  <!-- 内容 -->
</BaseLayout>
```

### 修改帖子解析逻辑
编辑 `src/lib/telegram/index.js` 中的 `parsePost()` 函数

### 添加新的媒体类型支持
在 `src/lib/telegram/index.js` 中找到媒体处理部分，添加新的选择器和处理逻辑

### 自定义样式
编辑 `src/assets/main.css`

## 部署

```bash
# 开发
pnpm dev

# 构建
pnpm build

# Docker
docker build -t channelpro .
docker run -p 4321:4321 -e CHANNEL=your_channel channelpro
```

**支持平台**: Vercel / Cloudflare Pages / Netlify / Docker / Node.js

## 设计决策

1. **零客户端 JS** - 全部服务端渲染，优化 SEO 和性能
2. **无数据库** - Telegram 即 CMS
3. **静态资源代理** - 避免 CORS 问题
4. **内存缓存** - 减少 API 调用
