# paper-snapshots — project notes

**Paper Notes**：我读过论文的中文图文海报合集。独立 GitHub Pages 项目站点
(`git@github.com:YufengJin/paper-snapshots.git`, 分支 `main`)，发布在
**`https://yufengjin.github.io/paper-snapshots/`**。纯静态 HTML/WebP，无构建步骤。
个人主页 `yufengjin.github.io` 页脚有入口链接。

## 布局
- 根 `index.html` — 合集落地页，**由 `pipeline/build_index.py` 生成，不要手改**。
- `<slug>/` — 每篇海报：`index.html`（中文正文 + WebP 图）+ `img/*.webp` + `meta.json`。
- `pipeline/` — `publish_to_site.sh`（发布）+ `build_index.py`（重建索引）+ `backfill_meta.py`。
- `.claude/skills/` — **属于项目、已纳入版本控制**：`paper-notes`（编排进料）+ `paper-poster`（单篇生成规范，含 `assets/poster-template.html`）。
- `_src/` — **git-ignored** 工作目录（`_inbox/` + `_processed/` + 原始图源）。可用 `$POSTERS_SRC` 覆盖。
- `.nojekyll` — 原样发布。

## 统一风格（与 learning-notes 一致，务必保持）
Claude 暖灰底 + 石墨灰强调，极简近单色。**不要再引入橙色或其它强调色。** 共用设计 token：

| 角色 | 值 | 角色 | 值 |
|---|---|---|---|
| 背景 `--bg` | `#F8F8F6` | 正文 `--ink` | `#201F1C` |
| 卡片/面板 `--panel` | `#FFFFFF` | 次级标题 | `#3A3833` |
| 次级面板/chip | `#F1EFEA` | 次要文字 `--muted` | `#6B675F` |
| 细边框 `--line` | `#E8E4DC` | 强调 `--acc` | `#5A5953` |

- 改色只动每篇 `index.html` 顶部 `:root{}` 的 token 与 `poster-template.html`、`build_index.py` 里的模板；**绝不手搓深色**。
- 语义色（warning 琥珀、error 红）可保留，但克制使用。

## meta.json（每篇海报的唯一数据源）
`{slug,title,arxiv_id,url,category,keywords[],summary_zh,date,source}`
- `category` = **4 大类之一（仅粗分组）**：`机器人 · Robotics` / `计算机视觉 · Computer Vision` / `生成模型 · Generative Models` / `理论与优化 · Theory & Optimization`。
- 细分主题靠 `keywords[]`（6–10 个），`build_index.py` 把它们变成可点击标签过滤器。
- 内容规范（`paper-poster` skill 强制）：正文中文，术语/指标/引文保留英文；数字逐字摘自论文，**不得编造**；头部+底部都要有 arXiv/来源链接；章节 = 动机/方法/实验/局限性。

## 添加论文（模型驱动）
```bash
cp paper.pdf _src/_inbox/        # 或截图 / .bib；或 echo "<id|url|title>" >> _src/_inbox/queue.txt
# 在本目录开 Claude 会话触发 paper-notes skill（“处理 paper inbox”）
#   → 逐篇生成到 _src/<slug>/，去重后自动跑 publish_to_site.sh 重建 index.html
git add . && git commit -m "add <slug>" && git push   # Pages 自动重新部署
```
- **按 arXiv id 去重**：生成前会 `grep` 现有 `meta.json`，已存在的跳过。

## 资产规则（保持仓库精简）
- **海报图一律 WebP**：`cwebp -q 80-85 -resize 1280-1600 0`（`publish_to_site.sh` 自动做并改写 `<img src>`）。
- **绝不提交原始大图 / PDF / 视频**；源文件留在 `_src/`（git-ignored）。
- 单文件远低于 ~9MB；大视频走外部托管，不进 git 历史。
- 上限是 GitHub Pages **1GB 构建站点**。

## 命名规范
- 目录 / 文件 / slug 一律 **小写 kebab-case**（`a-z 0-9 -`）：不用空格、大写、下划线或中文（URL 友好）。
- slug 短小、能认出论文（如 `pi05-vla`、`cot-vla`）；由 `paper-poster` skill 生成。
- 图片用描述性 kebab-case（如 `architecture.webp`、`teaser.webp`），统一放 `<slug>/img/`，扩展名小写 `.webp`。
- 中文只出现在页面标题/正文与 `meta.json` 文本字段里，**不进路径/文件名**。

## 其它约定
- 内部链接全部**相对路径**，保持与域名无关。
- 落地页是生成物，改标题/分类/关键词请改对应 `meta.json` 再重建，别直接编辑 HTML。
- 提交信息简洁；不提交 `_src/`、`.DS_Store`、本地 `.claude` 配置（仅 `.claude/skills/` 入库）。
