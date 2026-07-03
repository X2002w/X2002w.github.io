# blog - Hexo 博客项目

## 基本信息
- **博客地址**: https://X2002w.github.io
- **GitHub Pages 仓库**: git@github.com:X2002w/X2002w.github.io.git (master 分支)
- **作者**: liyan
- **GitHub**: https://github.com/X2002w
- **语言**: zh-CN

## 部署命令
```bash
hexo clean && hexo g && hexo d    # 清理 → 生成 → 部署
hexo clean && hexo g               # 仅重新生成（本地验证）
```

## 主题结构
- 主题从 `node_modules/hexo-theme-landscape` 复制到 `themes/landscape/` 进行自定义
- **不要修改 `node_modules/hexo-theme-landscape/`**，所有修改在 `themes/landscape/` 下
- Hexo 优先使用 `<项目>/themes/<主题名>/`，其次才用 `node_modules/` 中的主题

## 自定义模板文件 (`themes/landscape/layout/`)
| 文件 | 用途 |
|------|------|
| `layout.ejs` | 基础模板（HTML 框架、全局 CSS、Header、Footer） |
| `index.ejs` | 首页（Hero + 最近文章 + 标签云 + 热力墙） |
| `post.ejs` | 文章详情页 |
| `archive.ejs` | 归档页面 |
| `category.ejs` | 分类页面 |
| `tag.ejs` | 标签页面（标签云 + 单标签文章列表） |
| `page.ejs` | 静态页面模板 |

## 首页结构
1. **Hero 区域**: 头像（`/images/avatar.png`，可选）+ 副标题 + 两个按钮（博客列表 → /archives、GitHub → https://github.com/X2002w）
2. **最近文章**: 最新 5 篇，按日期降序
3. **标签云**: 最多 15 个标签
4. **热力墙**: 12 个月滚动窗口的 GitHub 风格贡献热力图

## 导航栏
- 只保留"首页"和"标签"两个链接
- 归档和分类的页面仍然存在（`/archives`、`/categories`），只是导航中不显示

## 热力墙（更新热力墙）实现细节
- **范围**: 固定 12 个月滚动窗口，以当天为终点。例如 2026-05-27 → 2025 年 6 月 ~ 2026 年 5 月
- **渲染**: 客户端 JavaScript。文章日期以 JSON 嵌入页面（`site.posts` → `postMap`），由 JS 动态生成热力图
- **着色逻辑**: 0 篇灰色、1 篇浅绿(#9be9a8)、2-3 篇中绿(#40c463)、4+ 篇深绿(#30a14e)
- **结构**: `.heatmap-outer` (flex) → `.heatmap-days` (固定星期列) + `.heatmap-wrapper` (可滚动区域)
- **星期标签**: Mon / Wed / Fri 三字母简写，固定在左侧不随热力墙滚动
- **月份标签**: 三字母英文简写（Jun ~ May），不标年份
- **初始位置**: 页面加载后自动滚动到最右侧（当天）
- **未来日期**: `display: none` 不占空间
- **标题**: 动态显示 `X posts in the past 12 months`

## 关键配置 (`_config.yml`)
```yaml
title: liyan
author: liyan
url: https://X2002w.github.io
language: zh-CN
theme: landscape
deploy:
  type: git
  repository: git@github.com:X2002w/X2002w.github.io.git
  branch: master
```

## 已知问题及修复
1. **Git dubious ownership**: `.deploy_git` 目录所有者不匹配时需要执行:
   ```bash
   git config --global --add safe.directory E:/workspace/blog/.deploy_git
   ```
2. **标签页 404**: 需要 `source/tags/index.md` 源文件（已创建，type: tags）

## 源文件结构
```
source/
├── _posts/           # Markdown 文章
│   ├── GTKWave-TCL-加载信号时找不到含有方括号的信号问题.md
│   └── Windows-RDP-连接-Debian-图形化桌面.md
└── tags/
    └── index.md      # 标签列表页（type: tags）
```

## 评论系统
- 支持三种评论系统：Disqus（第三方）、Valine（第三方 LeanCloud）、**Waline（自托管，首选）**
- 配置位于 `themes/landscape/_config.yml` 的 comment system 区域
- 如需禁用单篇文章的评论，在文章 front-matter 中添加 `comments: false`

### Waline（自托管方案）
- **数据存储**: 自己服务器上的 SQLite（Docker 部署，数据不经过第三方）
- **服务端**: Docker Compose 部署在 `/opt/waline/`，端口 8360，使用 SQLite
- **博客端**: Waline v3 客户端 JS 从 unpkg CDN 加载

### 评论系统相关文件
| 文件 | 作用 |
|------|------|
| `themes/landscape/layout/post.ejs` | 文章详情页底部评论容器（`#waline` / `.vcomment` / `#disqus_thread`） |
| `themes/landscape/layout/_partial/article.ejs` | 列表页评论数链接 + 非首页评论容器 |
| `themes/landscape/layout/_partial/after-footer.ejs` | 评论系统 JS 加载与初始化（Waline/Valine/Disqus） |
| `themes/landscape/_config.yml` | 评论系统配置（`disqus_shortname` / `valine` / `waline` 三段） |

### 启用 Waline 步骤
1. 服务器部署：`cd /opt/waline && docker compose up -d`
2. 修改 `themes/landscape/_config.yml` 中 `waline.enable: true`，填入 `serverURL`
3. 部署博客：`hexo clean && hexo g && hexo d`

## 页脚
- 仅显示 `© 2026 liyan.`（不包含 "Powered by Hexo"）
