# FAN.Design Library

> 搜狗设计团队的视觉灵感库。每天自动从 Behance / Dribbble / Pinterest / 数英 / Rebrand Gallery 抓取优质设计内容，沉淀为可浏览的灵感数据库，并为每条内容生成「搜狗输入法品牌/视觉启发」。

**网站地址：** https://gncoral.github.io/Design-Library/  
**GitHub 仓库：** https://github.com/gncoral/Design-Library  
**负责人：** nataliegao（搜狗输入法设计团队）  
**品牌色：** #3883FF

---

## 目录

- [整体架构](#整体架构)
- [数据结构](#数据结构)
- [每日自动任务](#每日自动任务)
- [脚本说明](#脚本说明)
- [内容过滤机制](#内容过滤机制)
- [sogouInsight 字段说明](#sogouinsight-字段说明)
- [环境变量 & Secrets](#环境变量--secrets)
- [AI 接手指南](#ai-接手指南)
- [常见问题排查](#常见问题排查)
- [Changelog](#changelog)

---

## 整体架构

```
GitHub Pages（静态网站）
    ↑ index.html 读取 data/*.json
    
GitHub 仓库（数据存储）
    ├── data/images.json       图片灵感库
    ├── data/digitaling.json   数英文章 + Rebrand 案例 + sogouInsight
    ├── data/blocked.json      用户叉掉的图（待清理）
    ├── data/liked.json        用户收藏的图
    ├── data/notes.json        用户备注
    ├── data/covers/dg/        数英封面图镜像
    ├── data/covers/mirror/    Dribbble/Pinterest 图片镜像
    └── data/uploads/          用户手动上传的文件

OpenClaw 云桌面（脚本运行环境）
    └── scripts/               每日自动任务脚本（Node.js）
        ↓ cron 定时触发
        → 抓取内容 → 调用 AI 生成 sogouInsight → git push → GitHub Pages 自动更新
```

**关键设计原则：**
- 网站托管在 GitHub Pages，完全独立，不依赖任何外部服务在线
- 所有数据存在 GitHub 仓库，脚本通过 GitHub API 或 git push 更新
- 脚本运行在 OpenClaw 云桌面，通过 cron job 定时触发
- AI 生成 sogouInsight 调用工蜂 AI API（`test-copilot.code.woa.com`）

---

## 数据结构

### `data/images.json`

图片灵感库，数组，**无数量上限，只增不减**（2026-05-11 已修复历史截断 bug）。

```json
[
  {
    "id": "d406",
    "title": "the message system that never misses",
    "src": "https://i.pinimg.com/736x/...",
    "url": "https://www.pinterest.com/pin/...",
    "source": "pinterest",      // "pinterest" | "dribbble" | "behance" | "upload" | "unknown"
    "category": "design",       // "branding" | "ui" | "design" | "illustration" | "web" | "motion"
    "tags": [],
    "added": "2026-04-15"
  }
]
```

**id 命名规则：**
- `d{数字}` — 自动抓取的图片（d1 起递增）
- `upload-{timestamp}` — 用户手动上传的文件

### `data/digitaling.json`

数英文章 + Rebrand Gallery 案例，数组，最新的在最前面。

```json
[
  {
    "id": "dg-2026-05-11",
    "date": "2026-05-11",
    "type": "daily",              // "daily" | "weekly"
    "source": "digitaling",       // "digitaling" | "rebrand"
    "sourceName": "数英",
    "sourceUrl": "https://www.digitaling.com/",
    "title": "文章标题",
    "url": "https://www.digitaling.com/articles/xxx.html",
    "coverImg": "https://gncoral.github.io/Design-Library/data/covers/dg/dg-2026-05-11.jpg",
    "summary": "文章摘要",
    "keyInsights": [],
    "tags": ["设计趋势", "数英精选"],
    "heat": { "zan": 29, "collect": 44, "comment": 6 },
    "sogouInsight": [
      "**标题加粗** 内容...",
      "**标题加粗** 内容...",
      "**标题加粗** 内容..."
    ]
  }
]
```

**Rebrand 类型额外字段：**
```json
{
  "source": "rebrand",
  "caseStudyUrl": "https://...",
  "companyUrl": "https://...",
  "industry": "Software",
  "designBy": "In-house",
  "visualStyles": ["Bold", "Gradient"],
  "typography": ["Sans Serif"]
}
```

### `data/liked.json`

```json
{
  "ids": ["d136", "d132", "upload-1776167827787"],
  "tagCounts": {},
  "categoryCounts": { "branding": 5 },
  "byUser": {
    "nataliegao": ["d136", "d132", "upload-1776167827787"]
  }
}
```

### `data/blocked.json`

用户叉掉但尚未清理的图片 ID 数组。每天 cleanup-blocked.js 运行后清空。  
**注意：被 liked 的图即使在 blocked 里也不会被删除（2026-05-11 加入保护逻辑）。**

```json
["d300", "d353"]
```

---

## 每日自动任务

所有任务运行在 OpenClaw 云桌面，通过 cron 调度，每天上午 10 点分批执行：

| 任务名 | 触发时间 | 脚本 | 说明 |
|--------|----------|------|------|
| `digitaling-daily-10:05` | 10:07 | `fetch-digitaling-auto.js` | 抓数英最热文章 + AI 生成搜狗启发 |
| `learn-from-blocked-daily` | 10:12 | `learn-from-blocked.js` | 分析叉图学习关键词，更新黑名单 |
| `behance-daily-10:00` | 10:23 | `cleanup-blocked.js` → `fetch-behance-rss.js` | 清理叉图 + 抓 8 张 Behance |
| `dribbble-daily-10:02` | 10:37 | `fetch-dribbble-playwright.js` | 抓 8 张 Dribbble |
| `rebrand-gallery-daily` | 10:43 | `rebrand-daily.js` | 抓 Rebrand Gallery 案例 + AI 生成搜狗视觉启发 |
| `pinterest-daily-10:04` | 10:51 | `fetch-pinterest-playwright.js` | 抓 8 张 Pinterest |
| `daily-summary-11:10` | 11:10 | *(AI 任务)* | 汇总今日更新状态，发到企微群 |

**每日正常产出：**
- Behance 0-8 张（RSS 过滤后可能较少）
- Dribbble 8 张
- Pinterest 8 张
- 数英文章 1 篇（含 3 条 sogouInsight）
- Rebrand 案例 1 个（含 2 条 sogouInsight）

---

## 脚本说明

### 核心生产脚本（每天跑）

#### `fetch-behance-rss.js`
从 Behance RSS 接口抓取图片，无需浏览器。
- 读取本地 `design-library/data/images.json`
- 抓取 5 个 RSS feed（branding / graphic-design / illustration / ui-ux / typography）
- 白名单 + 黑名单过滤标题
- 追加新图，**不截断历史**（`MAX_TOTAL = Infinity`）
- git push 更新

#### `fetch-dribbble-playwright.js`
用 Playwright 无头浏览器抓取 Dribbble popular shots。
- 抓取两页热门，过滤低质内容
- 将图片镜像到 `data/covers/mirror/`（防止 CDN 失效）
- 追加新图，**不截断历史**

#### `fetch-pinterest-playwright.js`
用 Playwright 搜索 Pinterest 设计类图片。
- 搜索词：`brand identity design minimal` / `saas ui design clean light` / `glass morphism ui design`
- alt 文字白名单过滤（必须含设计关键词）
- 将图片镜像到 `data/covers/mirror/`
- 追加新图，**不截断历史**

#### `fetch-digitaling-auto.js`
从数英 API 抓取每日精选文章。
- 调用 `/api/getJingArts`（编辑精选）和 `/api/getArticles/type/viewCount`（热门）
- 按收藏+点赞综合排序，取 top 1
- 抓文章封面图（og:image），用 Playwright 镜像到 `data/covers/dg/`
- 调用工蜂 AI API 生成 3 条搜狗品牌启发（sogouInsight）
- 写入 `digitaling.json`，git push
- **注意：此脚本 8 分钟 timeout，AI 生成步骤容易超时，超时后需手动补 sogouInsight**

#### `rebrand-daily.js`
从 rebrand.gallery 抓取当日品牌重塑案例。
- 维护已抓案例列表，每天取下一个未抓的
- 每周一额外生成本周合集（type: "weekly"）
- 调用工蜂 AI API 生成 2 条搜狗视觉设计启发
- 写入 `digitaling.json`，git push

#### `cleanup-blocked.js`
清理用户叉掉的图片。
- 读取 `blocked.json`，从 `images.json` 删除对应条目
- **收藏保护：** 被 `liked.json` 收藏的图即使在 blocked 里也不删除
- 清空 `blocked.json`
- git push

#### `learn-from-blocked.js`
自动学习用户审美偏好。
- 分析 `blocked.json` 里图片标题，提取高频词
- 自动追加到三个抓图脚本的黑名单里
- 维护 `learn-state.json` 记录已学关键词，有新词才通知
- 已学到的词：`logo animation`、`wip logo`、`neon brutalism`

### 手动工具脚本（按需使用）

| 脚本 | 用途 |
|------|------|
| `add_vw_entry.js` | 手动添加大众汽车相关条目 |
| `add-figma-rebrand.js` | 手动添加 Figma 品牌重塑案例 |
| `patch-sogou-today.js` | 手动补今日 sogouInsight |
| `generate-insight.js` | 单独生成某条内容的 sogouInsight |
| `check_digitaling.js` | 检查数英数据是否正常 |
| `find_next_case.js` | 查找下一个待抓的 Rebrand 案例 |
| `rebrand-today.js` | 手动触发今日 Rebrand 抓取 |
| `push-pending-entry.js` | 推送待发布条目 |

---

## 内容过滤机制

### Behance / Dribbble 黑名单（标题匹配）
```
mendelian, clinical, medical, genomic, statistical, random,
illustration, illust, cartoon, anime, character, mascot,
game, gaming, nft, 3d render, wallpaper, fan art, fanart,
children, kids, cute, kawaii, comic, sticker,
automotive, automobile, car launch, vehicle, motorcycle, ev launch,
photography, photo shoot, portrait, landscape, nature,
logo animation, wip logo, neon brutalism
```

### Behance / Dribbble 白名单（标题需含其一）
```
brand, branding, identity, logo, ui, app, saas, dashboard,
interface, product, web, mobile, landing, typography, packaging,
ai, tech, fintech, startup, visual identity, design system,
motion, graphic, editorial, poster, clean, minimal, light,
glass, gradient, texture, depth, material
```

### Pinterest 过滤
- **白名单**（alt 文字必须含其一）：brand / ui / minimal / layout / glass / gradient 等设计词
- **黑名单**：photo / car / nature / food / fashion / wedding 等非设计词
- **自动学习**：`learn-from-blocked.js` 每天分析叉图，自动追加黑名单

---

## sogouInsight 字段说明

每条数英文章生成 **3 条**搜狗输入法品牌启发，每条 Rebrand 案例生成 **2 条**视觉设计启发。

生成 prompt 背景：
> 搜狗输入法：大众级产品（数亿用户）、AI 能力持续升级、品牌年轻化、与其他输入法差异化竞争

格式：`**加粗标题** 内容...`，每条 100 字以内，直接可用于内部品牌/设计讨论。

**超时处理：** digitaling 任务容易超时（8分钟上限），sogouInsight 未生成时可手动运行：
```bash
node scripts/patch-sogou-today.js
```
或直接通过 AI 补写并 PUT 更新 `data/digitaling.json`。

---

## 环境变量 & Secrets

| 变量 | 用途 | 存放位置 |
|------|------|----------|
| `GITHUB_TOKEN` | 读写 GitHub 仓库内容 | 脚本内硬编码（cron 环境变量） |
| `GF_TOKEN` | 调用工蜂 AI API 生成 sogouInsight | `C:/openclaw/env.json` |
| `GF_USERNAME` | 工蜂 AI 用户名 | `C:/openclaw/env.json` |
| `CVM_AGENT_ID` | 工蜂 AI 设备 ID | `C:/openclaw/env.json` |

**工蜂 AI API endpoint：**  
`https://test-copilot.code.woa.com/server/openclaw/copilot-gateway/v1/chat/completions`  
使用模型：`claude-sonnet-4-6`

---

## AI 接手指南

### 项目状态速查

接手时先执行以下检查，3 步了解当前状态：

**1. 查今日 cron 运行状态**
```
调用 cron list，查看以下 6 个任务的 lastRunStatus 和 lastRunAtMs：
- digitaling-daily-10:05
- learn-from-blocked-daily
- behance-daily-10:00
- dribbble-daily-10:02
- rebrand-gallery-daily
- pinterest-daily-10:04
```

**2. 查今日 GitHub commits**
```
GET https://api.github.com/repos/gncoral/Design-Library/commits?per_page=10
用 GITHUB_TOKEN 认证，看今天日期的 commit 有哪些
```

**3. 查数据总量**
```
GET https://api.github.com/repos/gncoral/Design-Library/contents/data/images.json
解码 base64 content，检查：
- 总数量（应持续增长，不应被截断）
- 各 source 分布：pinterest / dribbble / behance / upload
- liked.json 里的 id 是否都能在 images.json 中找到
```

### 常见任务

**手动补 sogouInsight（digitaling 超时时）**
1. 获取今日 `digitaling.json` 中 source=digitaling、sogouInsight 为空的条目
2. 根据 title 和 summary 生成 3 条启发（参考上方 prompt 背景）
3. PUT 更新 `data/digitaling.json`，写入 sogouInsight 字段

**手动触发某个 cron 任务**
```
cron run <jobId>
```

**恢复丢失图片**
```
从 git log 找历史版本：
git show <commit>:data/images.json
对比当前版本，提取丢失的条目
PUT 合并回 data/images.json
```

**检查收藏是否丢失**
```
liked.json 中的 ids 需在 images.json 中都能找到
若找不到，需从 git 历史或手动补充占位条目
```

### 脚本运行环境

- **运行机器：** OpenClaw 云桌面（Windows）
- **脚本目录：** `C:\Users\Administrator\.openclaw\workspace\scripts\`
- **仓库目录：** `C:\Users\Administrator\.openclaw\workspace\design-library\`
- **Node.js 版本：** v22
- **依赖：** playwright（已安装）

**运行任意脚本：**
```powershell
cd C:\Users\Administrator\.openclaw\workspace\scripts
$env:GITHUB_TOKEN="github_pat_..."; node fetch-behance-rss.js
```

---

## 常见问题排查

### 图片数量不增加 / 旧图消失
**原因：** 脚本中 `MAX_TOTAL` 常量截断了历史数据（已于 2026-05-11 修复，所有脚本改为 `Infinity`）  
**排查：** 检查三个抓图脚本是否还有 `updated = updated.slice(excess)` 类似代码  
**修复：** 从 git 历史恢复丢失条目，参考「恢复丢失图片」步骤

### 收藏数显示 0
**原因：** `liked.json` 里的 id 对应的图片已从 `images.json` 中删除  
**修复：** 把缺失的 id 对应条目补回 `images.json`（可用占位图）

### digitaling 任务每天超时
**原因：** 一个 job 里串行做「爬文章」+「AI 生成 sogouInsight」，总时长超过 8 分钟  
**临时方案：** 超时后手动补 sogouInsight  
**根本方案：** 拆成两个 cron job，或将 timeoutSeconds 调大到 720

### Behance 抓不到图
**原因：** Behance RSS 对 `api_key=test` 限流，或白名单过滤过严  
**排查：** 运行 `node test-behance-rss.js` 查看 RSS 返回内容

### Dribbble / Pinterest 图片加载失败
**原因：** CDN 防盗链  
**解决：** 脚本已将图片镜像到 `data/covers/mirror/`，前端优先读镜像地址

### git push 失败
**原因：** 多个 cron job 并发写同一文件产生冲突  
**解决：** 脚本内置 `git pull --rebase` + 3 次重试逻辑，通常自动恢复

---

## Changelog

### [2026-05-11] — 数据稳定性大修

#### 🔴 重大 Bug 修复
- **恢复 88 张历史图片**：`MAX_TOTAL` 限制导致从 2026 年 3 月起的图片被持续裁剪丢失，今日从 git 历史完整恢复
- **images.json 不再有上限**：三个抓图脚本（Behance / Dribbble / Pinterest）`MAX_TOTAL` 从 200/300 改为 `Infinity`，只增不减
- **收藏图保护**：`cleanup-blocked.js` 新增逻辑，被 `liked.json` 收藏的图即使被叉也不会删除
- **补回收藏和上传**：d136、d132、upload-1776167827787 因截断丢失，已补回 images.json

#### ✅ 其他
- 手动补今日数英 sogouInsight（任务超时未生成）
- 生成本 README 文档

---

### [2026-04-17] — 内容过滤加强 + 自动审美学习

#### 🧹 内容质量
- **清除跑偏图片**：移除汽车广告图（NIO ES9 Launch Campaign）和植物摄影图，共 2 张
- **Behance / Dribbble 黑名单加强**：新增 `automotive`、`car launch`、`vehicle`、`motorcycle`、`photography`、`photo shoot` 等词
- **Pinterest 新增白名单过滤**：alt 文字必须包含设计类关键词才收录

#### 🧠 自动审美学习
- **新增 `learn-from-blocked.js`**：每天 10:12 自动运行，分析叉图提取规律，追加黑名单
- 首次已学到：`logo animation`、`wip logo`、`neon brutalism`

#### 🐛 Bug 修复
- **卡片显示 `undefined`**：兼容 `source` 和 `platform` 字段，修复来源标签渲染

---

### [2026-04-14] — 品牌升级 + 稳定性修复

#### 🎨 品牌与视觉
- **重命名**：网站更名为 `FAN.Design Library`
- **品牌色**：全站换为 #3883FF，按钮/边框/高亮同步更新
- **Header 简化**：移除顶部平台/标签筛选条

#### ✨ 新功能
- **视频上传支持**：支持 MP4/WebM，自动播放卡片展示
- **叉图 Toast 随机文案**：8 条有个性的反馈文案

#### 🔧 Bug 修复
- **图片点击放大**：改为事件委托彻底修复
- **模板字符串嵌套 Bug**：makeCard 里视频/图片判断改为字符串拼接
- **页面初始化崩溃**：清理残留 JS 事件监听
- **外链图片防盗链**：加入 `images.weserv.nl` 代理中转

#### 🌀 加载体验
- **Loading Spinner**：蓝色旋转动画 + "灵感加载中…"

---

> 更早记录见 git log：`git log --oneline -- data/images.json`
