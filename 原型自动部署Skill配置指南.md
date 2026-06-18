# deploy-prototype Skill 配置文档

> 目标：让其他 AI（Claude Code 等）读完本文档后，能从零配置出与原版完全一致的 skill，并无缝接入工作流。

---

## 一、这个 Skill 做什么

当用户说"部署原型"、"推到 GitHub"或刚刚创建/修改了 HTML 原型文件时，AI 自动执行：

1. 将本地**中文命名**的 HTML 文件映射为**英文文件名**
2. 复制为英文文件名，推送到 GitHub Pages
3. 返回两个链接：本地预览（`file://`，即时可用）和云端链接（`https://`，数分钟后生效）

**核心设计原则**：本地保留中文文件名方便辨认，云端只有干净的英文 URL 便于分享。

---

## 二、前置准备（一次性操作）

### 2.1 创建 GitHub 仓库

1. 在 GitHub 新建一个公开仓库，例如 `your-username/prototype-pages`
2. 进入仓库 Settings → Pages，Source 选 `main` 分支根目录，保存
3. 等待几分钟，`https://your-username.github.io/prototype-pages/` 即可访问

### 2.2 创建本地目录并初始化 git

```bash
mkdir ~/prototype_html
cd ~/prototype_html
git init
git remote add origin https://github.com/your-username/prototype-pages.git
```

### 2.3 创建 `.gitignore`

作用：默认排除所有 HTML，仅允许英文命名的文件上传。

```
.DS_Store

# 中文命名的原型文件只保留在本地，不上传 GitHub
# 英文文件（通过 name-map.json 对应）才是云端版本
*.html
!index.html
```

> 每次新增一个英文文件，都要在此文件末尾追加一行 `!英文文件名.html`

### 2.4 创建 `name-map.json`

作用：维护"中文文件名 → 英文文件名"的映射关系。

```json
{}
```

初始为空对象即可，AI 在部署时会自动维护。

### 2.5 首次提交

```bash
git add .gitignore name-map.json
git commit -m "init: 初始化原型部署仓库"
git push -u origin main
```

---

## 三、Skill 文件配置

在 Claude Code 的 skills 目录下创建以下结构：

```
~/.claude/skills/deploy-prototype/SKILL.md
```

### SKILL.md 内容

将下方内容**完整复制**到文件中，并替换所有 `<占位符>` 为实际值：

```markdown
---
name: deploy-prototype
description: 将本地中文命名的 HTML 原型文件同步到 GitHub Pages（英文文件名），并返回可访问的页面链接。当用户说"部署原型"、"推到 GitHub"、"更新 GitHub Pages"或者刚创建/修改了 HTML 原型文件时调用。
user-invocable: true
---

# deploy-prototype

将本地中文命名的 HTML 原型文件部署到 GitHub Pages。

## 基本信息

- 本地原型目录：`<本地目录，如 ~/prototype_html/>`
- 仓库：`https://github.com/<用户名>/<仓库名>`
- Pages 前缀：`https://<用户名>.github.io/<仓库名>/`
- 映射文件：`<本地目录>/name-map.json`

## 规则

- **本地**保留中文文件名（被 `.gitignore` 排除，永远不上传）
- **云端（GitHub Pages）** 只有英文文件名，URL 干净可分享
- `name-map.json` 维护中文 → 英文的对应关系

## 执行步骤

收到部署请求后，按以下步骤执行：

### 1. 确认目标文件

如果用户指定了文件名，直接使用。否则询问要部署哪个文件。

### 2. 查找英文文件名

读取 `name-map.json`，找到对应的英文文件名。

**如果没有映射**，按命名规范新建一条：
- 小写英文 + 连字符
- 直接描述页面内容，不加 `admin-`/`c-` 等前缀
- 例：`B端_月卡基础信息配置_v2.html` → `monthly-card-basic-config-v2.html`
- 将新映射写入 `name-map.json`

### 3. 同步文件内容

```bash
cd <本地目录>
cp "中文文件名.html" "英文文件名.html"
```

### 4. 提交并推送

```bash
git add 英文文件名.html
# 如果 name-map.json 有改动也一并 add
git add name-map.json .gitignore
git commit -m "deploy: 更新 <页面描述>"
git push origin
```

### 5. 返回链接

同时返回两个链接：

**本地预览**（即时可用）：
```
file://<本地目录>/<中文文件名>.html
```

**云端地址**（用于贴文档，有几分钟延迟）：
```
https://<用户名>.github.io/<仓库名>/<英文文件名>.html
```

## 注意事项

- 永远不要 `git add` 中文命名的文件
- **新文件必须同时做三件事**：更新 `name-map.json`、在 `.gitignore` 末尾追加 `!英文文件名.html`、git add 三个文件后一起提交
- 如果用户同时更新了多个文件，批量 cp + git add 后一次提交
```

---

## 四、与工作流的衔接规则

配置好 skill 后，AI 需要遵循以下行为规范，才能"无缝衔接"：

### 4.1 自动触发时机

| 场景 | 期望行为 |
|------|----------|
| 用户说"部署原型" / "推到 GitHub" | 立即执行 deploy-prototype |
| 用户说"更新 GitHub Pages" | 立即执行 deploy-prototype |
| AI 刚生成/修改了 HTML 原型文件 | **主动询问**"要部署到 GitHub Pages 吗？" |

### 4.2 每次交付 HTML 时必须提供两个链接

```
本地预览：file:///Users/.../prototype_html/中文文件名.html
云端地址：https://your-username.github.io/repo-name/英文文件名.html
```

不能只给其中一个。本地链接用于即时预览，云端链接用于贴到文档/分享。

### 4.3 文件命名规范

- 中文文件名：保留在本地，自由命名，方便人类识别
- 英文文件名：小写 + 连字符，语义直接，不加模块前缀

---

## 五、配置验证

配置完成后，执行以下检查确认一切就绪：

```bash
# 1. 确认本地目录存在
ls ~/prototype_html/

# 2. 确认 git remote 正确
cd ~/prototype_html && git remote -v

# 3. 确认 name-map.json 可读
cat ~/prototype_html/name-map.json

# 4. 确认 GitHub Pages 已开启（浏览器访问）
# https://your-username.github.io/repo-name/
```

如果以上全部通过，skill 即可正常工作。

---

## 六、当前原版配置参数（供参考）

| 参数 | 值 |
|------|----|
| 本地目录 | `~/prototype_html/` |
| GitHub 仓库 | `javenvian-del/lcp-prototype-pages` |
| Pages 根 URL | `https://javenvian-del.github.io/lcp-prototype-pages/` |
| 映射文件 | `~/prototype_html/name-map.json` |
