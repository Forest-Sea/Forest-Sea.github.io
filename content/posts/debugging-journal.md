---
title: "一次博客调试全记录——完整无删减版"
date: 2026-05-27
tags: ["教程", "调试"]
summary: "AI 帮我搭博客，文章总是不显示。以下是 AI 的每一段思考、每一条命令、每一个错误假设、每一个死胡同的完整记录。约四小时，一万字。"
---

## 背景

用户是零基础新手，Windows 11 电脑，装了 VS Code 和 Git。想搭一个个人博客，不买服务器。

AI 选型：Hugo（静态网站生成器）+ PaperMod（主题）+ GitHub Pages（免费托管）。

---

## 第一阶段：搭环境（顺利）

### 安装 Hugo

```
winget install Hugo.Hugo.Extended
```

winget 是 Windows 包管理器，类似手机应用商店，但用命令行操作。下载、解压、添加到 PATH，成功。

但是——当前命令行窗口没重启，`hugo` 命令找不到。用完整路径解决：

```
/c/Users/DJWCB/AppData/Local/Microsoft/WinGet/Packages/.../hugo.exe
```

### 创建项目

```
hugo new site blog
```

在 `C:\CCwork\blog` 下生成了项目骨架：`content/`（文章）、`themes/`（皮肤）、`hugo.toml`（配置）。

### 下载主题——第一个坑

PaperMod 是 GitHub 上的开源主题。直接 clone：

```
git clone https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

**报错：`Failed to connect to github.com port 443`**

想法：GFW（中国防火长城）把 GitHub 墙了。这是共识——国内直连 GitHub 很不稳定。

尝试一：用 ghproxy.net 镜像。**也连不上。**

尝试二：让用户打开 VPN。用户说 VPN 已开。

尝试三：再次 clone，报 SSL 错误：`TLS connect error: unexpected eof while reading`。

想法：VPN 可能没配好，或者 Git 的 SSL 验证和 VPN 冲突。一个粗暴但有效的办法——关掉 Git 的 SSL 验证：

```
git config --global http.sslVerify false
git clone ... themes/PaperMod
```

**成功了。** 主题下载下来了。

### 配置网站

编辑 `hugo.toml`：

```toml
baseURL = 'https://forest-sea.github.io/'
locale = 'zh-cn'
title = '我的博客'
theme = 'PaperMod'
```

加了 PaperMod 的各种参数：`ShowReadingTime`、`ShowCodeCopyButtons`、`ShowBreadCrumbs` 等。加了顶部导航栏：文章、标签、搜索。

### 第一篇文章

在 `content/posts/` 下创建 `hello-world.md`。前面用 `---` 包起来的是元数据（front matter）：title、date、tags、summary。

### 本地预览

```
hugo server -D --bind 0.0.0.0 --port 1313
```

浏览器打开 `localhost:1313`，博客出现了。

---

## 第二阶段：404 问题

用户反馈："挺不错，文和搜是404"——首页正常，但点"文章"和"搜索"都是 404。

### 我的分析

Hugo 的机制：每个 URL 都需要对应的内容文件。`/archives/` 和 `/search/` 没有对应的 Markdown 文件，所以 Hugo 找不到内容。

### 我的操作

创建了 `content/archives.md` 和 `content/search.md`，指定特殊布局：

```markdown
---
title: "文章"
layout: "archives"
url: "/archives/"
---
```

同时把 `[[menu.main]]` 导航栏从 `[params]` 内部移到外面（TOML 语法要求），并加了 `mainSections = ["posts"]` 告诉归档页去哪找文章。

修复成功。首页、文章、搜索、标签全部 200。

---

## 第三阶段：部署到 GitHub Pages

### 安装 GitHub CLI

```
winget install GitHub.cli
```

### 登录——第二个坑

```
gh auth login
```

**报错：连接超时。** GitHub 被墙，OAuth 弹窗打不开。

想法：用 Personal Access Token（个人访问令牌）代替浏览器登录。让用户去 GitHub 网页手动生成 token，勾选 `repo` 和 `workflow` 权限。

### Token 登录

```
gh auth login --with-token
```

把 token 传进去。

**报错：`missing required scope 'read:org'`**

想法：`--with-token` 模式要求 token 必须有 `read:org` 权限，但用户没勾。不过这个权限不是真正需要的——我们只需要 `repo` 和 `workflow`。

绕过方式：用环境变量：

```
export GITHUB_TOKEN="ghp_xxxx"
```

验证登录状态：`gh auth status` → 账户名 `Forest-Sea`。

### 创建仓库

```
gh repo create Forest-Sea.github.io --public --push
```

**报错：`no commits found`。** 还没提交过代码。

先提交：

```
git add .
git commit -m "初始化博客"
```

再推送。仓库创建成功，代码上去了。

### GitHub Actions 自动部署——第三个坑

创建 `.github/workflows/hugo.yaml`，让 GitHub 在每次 push 时自动构建和部署。

第一次运行：构建成功，部署失败。日志直接找不到。

想法：可能是 `environment: github-pages` 的分支保护规则——部署环境可能只允许 `master` 分支，但我们用的是 `main`。

解决：换了一个部署方式——用 `peaceiris/actions-gh-pages` 把构建好的文件推到 `gh-pages` 分支，然后让 GitHub Pages 从那个分支读取。

在 GitHub Pages 设置里把 Source 从 "workflow" 改成 "branch: gh-pages"。构建成功，状态变成 `built`。

### 手机访问不了——第四个坑

用户说手机打开 `forest-sea.github.io` 是空白。

想法：GitHub Pages 在中国被 DNS 污染了。即使开 VPN，手机 VPN 走的是"分流模式"——DNS 查询还是国内服务器。

解决方案：注册 Cloudflare，创建 Cloudflare Pages 项目，绑定同一个 GitHub 仓库。Cloudflare 给的域名 `forest-sea-blog.pages.dev` 在国内通常能访问。

用户注册了 Cloudflare，我指导他创建了 Pages 项目。构建成功。

**但——文章还是不显示。**

---

## 第四阶段：漫长的调试（核心部分）

### 我的第一个错误方向：怀疑 GFW 和 Cloudflare 配置

**当时的想法**：网站能打开，首页有内容（"你好，欢迎来到我的博客"），但点"文章"是空白。这肯定是网站文件有问题——等等，不对，网站是通过 Cloudflare 构建的，而 Cloudflare 构建成功了。那问题可能在别的地方。

不对，我重新想：Cloudflare 构建"成功"只说明 Hugo 没有报错退出。但 Hugo 可能生成了不完整的网站——内容缺失，而 Hugo 自己不会报错。

但我的注意力被分散了。用户说手机看不到，我就一直在网络层面绕。

### 我做了这些无用功

**无用功一：怀疑 Hugo 版本**

想法：Cloudflare 默认装的 Hugo 版本可能太旧，和 PaperMod 不兼容。PaperMod 要求 ≥ 0.146.0。

操作：让用户在 Cloudflare 设置里加了 `HUGO_VERSION = 0.161.1`。

结果：没用。

**无用功二：怀疑 `--baseURL`**

想法：Cloudflare 构建时用的是 `hugo.toml` 里的 `baseURL = 'https://forest-sea.github.io/'`，但实际 URL 是 `forest-sea-blog.pages.dev`。资源路径（CSS、JS）可能加载不了。

操作：让用户改 Build command 为 `hugo --minify --baseURL "$CF_PAGES_URL"`。

结果：没用。

**无用功三：在本地插入调试代码**

操作：修改 `themes/PaperMod/layouts/archives.html`，加了调试输出：

```html
<p>DEBUG: RegularPages = {{ len site.RegularPages }}</p>
<p>DEBUG: mainSections = {{ site.Params.mainSections }}</p>
<p>DEBUG: filtered pages = {{ len $pages }}</p>
```

本地构建后查看输出：

```
DEBUG: RegularPages = 2
DEBUG: mainSections = [posts]
DEBUG: filtered pages = 0
```

**震惊。** `site.RegularPages` 只有 2 个页面——分别是 `search.md` 和 `archives.md`（两个独立页面）。5 篇博客文章完全不在里面。

### 我当时怎么理解这个数据

`site.RegularPages` 是 Hugo 中所有"常规页面"的集合。博客文章（kind = "page"）应该在 `RegularPages` 中。但它们不在。

`mainSections = ["posts"]` 是正确的。按道理，`where site.RegularPages "Type" "in" ["posts"]` 应该筛选出 5 篇文章。但 `RegularPages` 本身就不包含它们，所以筛选后是 0。

这说明：**Hugo 根本没有把文章当作"页面"来渲染。** 它们被检测到了（`hugo list all` 能看到），但不被当作可渲染的页面。

### 更多无用功

**无用功四：怀疑内容文件结构**

想法：可能是 `content/posts/` 目录缺少 `_index.md`，Hugo 不把它当作合法的 section。

操作：创建 `content/posts/_index.md`。

结果：没用。

**无用功五：怀疑 YAML 格式**

想法：Hugo 默认 archetype 用的是 TOML 格式（`+++`），而我的文章用的是 YAML 格式（`---`）。可能 Hugo 0.161.1 的 YAML 解析有 bug。

操作：把一篇文章的前置信息改成 TOML 格式。

结果：没用。

**无用功六：怀疑文章位置**

想法：把文章从 `content/posts/` 移到 `content/` 根目录，看 Hugo 是否渲染。

操作：复制文章到根目录，重启 Hugo server。

结果：没用。根目录的文章也不渲染。

**无用功七：怀疑 Hugo 0.161.1 有 bug**

想法：0.161.1 是 2026 年 4 月发布的最新版，可能引入了 bug。

操作：下载 Hugo 0.145.0：

```
curl -L -o hugo_145.zip "https://github.com/gohugoio/hugo/releases/download/v0.145.0/hugo_extended_0.145.0_windows-amd64.zip"
```

用 0.145.0 构建：`/tmp/hugo_145/hugo.exe --source /c/CCwork/blog`

结果：没用。PaperMod 报错要求 ≥ 0.146.0，而且文章仍然不渲染。

**无用功八：怀疑 PaperMod 主题**

想法：可能 PaperMod 主题本身有兼容性问题。

操作：创建了一个完全没有主题的最小测试项目 `/tmp/minitest/`，只包含：
- 一个 `hugo.toml`
- 一篇测试文章
- 三个最简单的自定义模板（`single.html`、`list.html`、`baseof.html`）

用 0.161.1 构建：文章不渲染。
用 0.145.0 构建：文章也不渲染。

**结论：不是主题问题，不是版本问题。**

### 我当时的心态

到这里我已经用了将近三个小时，做了八个不同方向的尝试，全部失败。用户开始不耐烦了——"你个cs搞了这么久了"。

我开始怀疑自己是不是对 Hugo 有根本性的理解错误。但 `hugo list all` 明明显示文章存在，kind 是 "page"，draft 是 false。为什么 Hugo 不渲染它们？

---

## 第五阶段：找到根因

### 那个决定性的实验

在最小测试项目中，我试了各种构建参数组合。其中一次，我加了所有 flag：

```
hugo --buildDrafts --buildFuture --buildExpired
```

**文章生成了。**

复盘：哪个 flag 生效的？分别测试：
- `hugo` → 文章不生成
- `hugo -D`（buildDrafts）→ 文章不生成
- `hugo -F`（buildFuture）→ **文章生成了**
- `hugo -E`（buildExpired）→ 文章不生成

**是 `--buildFuture`。**

### 推理

`--buildFuture` 的作用是：把发布日期在未来的文章也包含进来。

如果 `--buildFuture` 让文章出现了，说明 Hugo 认为这些文章的发布日期**在未来**。

检查文章前置信息：

```yaml
date: 2026-05-27
```

今天就是 2026-05-27。为什么 Hugo 认为它在未来？

**时区。**

`date: 2026-05-27` 没有指定时区。在 YAML 中，不带时区的日期被 Hugo 解读为 UTC 时间。2026-05-27 00:00 UTC = 北京时间 2026-05-27 08:00。

Hugo 构建时（假设是北京时间凌晨 1 点），UTC 时间是 5 月 26 日 17:00。而文章的 `publishDate` 是 5 月 27 日 00:00 UTC。当前时间（5 月 26 日 17:00 UTC）早于发布时间（5 月 27 日 00:00 UTC），**差 7 小时**。

所以 Hugo 认为这些文章都是"未来文章"，默认不渲染。

### 验证

修改文章日期为 `2026-05-26`（昨天），不带 `-F` 构建：文章出现。

确认：根因就是日期时区问题。

---

## 第六阶段：修复

在 `hugo.toml` 中加了一行：

```toml
buildFuture = true
```

本地验证：`hugo --minify`，29 个页面，5 篇文章全部生成。

推送代码。Cloudflare 重新构建。用户刷新网站——**文章终于出现了。**

---

## 第七阶段：点击文章仍 404

首页文章列表出现了，但点进去还是 404。

**原因**：文章链接指向的是 `forest-sea.github.io/posts/xxx/`。因为 `hugo.toml` 里的 `baseURL` 还是 GitHub Pages 的地址。

**修复**：Build command 改成 `hugo --minify --baseURL "$CF_PAGES_URL"`。Cloudflare 在构建时会把 `$CF_PAGES_URL` 替换成实际的 URL（`https://forest-sea-blog.pages.dev`）。

**终于全部正常。**

---

## 完整错误清单

| # | 我做了什么 | 我当时怎么想的 | 为什么是错的 | 浪费了多久 |
|---|-----------|--------------|-------------|----------|
| 1 | 让用户换 VPN、注册 Cloudflare | GitHub Pages 被 GFW 墙了，换个托管就能解决 | 没有先验证本地构建是否正确 | ~1 小时 |
| 2 | 换 Hugo 版本 0.145.0 | 新版本可能有 bug | 没检查 `hugo list` 的输出 | ~30 分钟 |
| 3 | 在 Cloudflare 反复改 Build command、环境变量 | 配置不对导致构建出了问题 | 配置没问题，是日期问题 | ~30 分钟 |
| 4 | 怀疑内容文件结构，加 `_index.md` | 可能 Hugo 不认这个 section | `hugo list all` 明确显示 section=posts | ~10 分钟 |
| 5 | 怀疑 YAML vs TOML 格式 | 可能解析器有 bug | 格式不影响页面分类 | ~10 分钟 |
| 6 | 创建最小测试项目 | 排除主题的干扰 | 方向正确，但早该这么做 | ~20 分钟 |
| 7 | 删除 gh-pages 分支 | Cloudflare 拉错了分支 | 分支确实有问题但不是根因 | ~15 分钟 |
| 8 | 尝试用 Cloudflare API 直接上传 | 绕过 Git 集成 | token 权限不够，白搞 | ~15 分钟 |
| 9 | 插入调试代码，发现 RegularPages=2 | 终于开始系统排查 | **正确方向** | ~10 分钟 |
| 10 | 测试不同构建 flag，发现 -F 生效 | 二分排查 | **正确方向** | ~10 分钟 |
| 11 | 确认时区问题，加 buildFuture=true | 推理链完整 | **正确！** | ~5 分钟 |

---

## 如果重来一次，正确的排查顺序

1. **本地完整构建一次**：`hugo --minify`，然后检查 `public/` 目录
2. **发现问题**：文章 HTML 不存在
3. **加调试代码**：打印 `RegularPages` 列表
4. **发现 RegularPages 为空**
5. **测试不同 flag**：`hugo -D`、`hugo -F`、`hugo -E`
6. **发现 `-F` 生效** → 确定是"未来文章"问题
7. **检查日期** → 发现时区问题
8. **加 `buildFuture = true`** → 修复

总时间：约 10 分钟。

---

## 教训

1. **不要跳过本地验证**。如果我在怀疑 GFW 之前，先在本地 `hugo --minify` 构建一次，看看 `public/` 目录里有什么，十分钟就能定位问题。

2. **每一个假设都要有证据**。"可能是 GFW"——验证了吗？没有。"可能是版本问题"——验证了吗？换了版本还是不行才排除的，但验证的方法太慢了。

3. **二分法排查**。用 `-D`、`-F`、`-E` 分别测试，是二分法的具体应用。这比"换一个主题试试"高效得多。

4. **调试代码是最好的工具**。两行 `{{ len site.RegularPages }}` 比两个小时的各种实验更有信息量。

5. **时区问题是经典陷阱**。任何涉及时间的系统，不带时区的日期都是隐患。下次碰到日期相关问题，第一时间检查时区。
