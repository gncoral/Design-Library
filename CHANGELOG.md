# Changelog

## [2026-04-17] — 内容过滤加强 + 自动审美学习

### 🧹 内容质量
- **清除跑偏图片**：移除汽车广告图（NIO ES9 Launch Campaign）和植物摄影图（shadow of a plant），共 2 张
- **Behance / Dribbble 黑名单加强**：新增 `automotive`、`car launch`、`vehicle`、`motorcycle`、`photography`、`photo shoot` 等词，拦截非设计类内容
- **Pinterest 新增白名单过滤**：抓图时 alt 文字必须包含设计类关键词（brand / ui / minimal / layout 等）才收录；同时新增黑名单拦截 photo / car / nature / food / fashion 等类目

### 🧠 自动审美学习
- **新增 `learn-from-blocked.js`**：每天 10:12 自动运行，分析你叉掉的图片标题，提取规律关键词，自动追加到三个抓图脚本的黑名单里
- 首次运行已从历史叉图中学到：`logo animation`、`wip logo`、`neon brutalism`
- 有新学到的内容才通知，无变化静默

### 🐛 Bug 修复
- **卡片显示 `undefined`**：修复图片来源标签渲染问题——旧数据用 `source` 字段，前端却读 `platform` 字段导致显示 undefined，已兼容两个字段，无来源则隐藏标签

---

## [2026-04-14] — 品牌升级 + 稳定性修复

### 🎨 品牌与视觉
- **重命名**：网站更名为 `FAN.Design Library`，标题栏、logo、欢迎弹窗同步更新
- **品牌色**：全站颜色由红色 `#ff6b6b` 换成品牌蓝 `#3883FF`（按钮、边框、高亮、通知点等），收藏心形保留红色
- **Header 简化**：移除顶部平台/标签筛选条，页面更干净
- **Tab 放大**：导航 Tab 字号从 13px → 16px，间距加宽，去掉 emoji 前缀

### ✨ 新功能
- **视频上传支持**：支持 MP4/WebM 视频上传，在灵感库以自动播放卡片展示（muted loop）
- **视频上传预览**：选文件后先显示「⏳ 读取中…」，可播放后显示「文件名 · X MB ✅」，格式不支持时给出提示
- **叉图 Toast 随机文案**：叉掉作品时随机显示 8 条有个性的反馈文案（如"📐 收到，正在校准审美坐标…"）

### 🔧 Bug 修复
- **图片点击放大修复**：`onclick` 写在 HTML 属性里引号冲突导致失效，改为事件委托（`class="card-img"` + grid 监听），彻底修复
- **模板字符串嵌套 Bug**：`makeCard` 里视频/图片三元判断使用嵌套反引号导致变量全变 `undefined`，改为字符串拼接
- **页面初始化崩溃**：删除 header filter 条后漏删对应 JS 事件监听（`getElementById('filters')`），导致 `init()` 中断、图片不显示，已清理
- **外链图片防盗链**：Dribbble/Behance/Pinterest 图片在不同网络环境下加载不稳定，加入 `images.weserv.nl` 代理中转，提升跨设备一致性

### 🌀 加载体验
- **Loading Spinner**：页面初次加载时显示蓝色旋转动画 + "灵感加载中…"，数据就绪后自动消失，不再出现空白骨架屏
- Spinner 位置修正：从 grid 容器内移至外部独立居中，圆圈形状正确

---

## 历史版本

> 更早的更新记录请查看 git log
