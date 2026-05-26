---
title: "从零搭建个人博客全记录——每一步、每个报错、每个修复"
date: 2026-05-27
tags: ["教程", "博客搭建"]
summary: "完整流水账——一台 Windows 电脑、零基础、不花一分钱搭建个人博客的全程实录，含所有踩坑和修复。"
---

## 起点

我想搭一个个人网站，但不买服务器，不花钱。

工具：一台 Windows 11 电脑，装了 VS Code，连了 VPN，其他什么都没有。

最终方案：**Hugo（静态网站生成器）+ PaperMod（主题）+ GitHub Pages（免费托管）**。

下面是一步不漏的全过程。

---

## 一、装备检查

先看电脑上有什么：

```bash
git --version    # 输出了 2.53.0，Git 已装
winget --version # 输出了 1.28.240，Windows 包管理器已有
```

**git** 是版本管理工具——记录你对代码的每一次修改，能把代码推到 GitHub。

**winget** 是 Windows 自带的命令行"应用商店"，能一键安装软件。

## 二、安装 Hugo

```bash
winget install Hugo.Hugo.Extended
```

**Hugo 是什么？** 静态网站生成器。你用 Markdown 写文章，它帮你拼成完整的网站（HTML 文件）。

下载的是一个 zip 包，winget 自动解压、添加到系统路径。安装成功。

但是！当前命令行窗口没有重启，输入 `hugo` 报错 `command not found`。因为环境变量还没刷新。

解决方法：找到 Hugo 的完整路径，直接用它：

```bash
/c/Users/DJWCB/AppData/Local/Microsoft/WinGet/Packages/.../hugo.exe
```

## 三、创建博客项目

```bash
hugo new site blog
```

这条命令在 `C:\CCwork\blog` 下生成了一整套项目骨架：

- `content/` — 放文章的地方
- `themes/` — 放皮肤的地方
- `hugo.toml` — 网站配置文件（标题、网址等）

## 四、初始化 Git 仓库

```bash
cd blog
git init
```

把这个文件夹变成 Git 仓库。后面要把代码传到 GitHub，必须要有这步。

## 五、下载主题——第一个坑

PaperMod 是一个流行的 Hugo 主题，简洁好看。从 GitHub 下载：

```bash
git clone https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

**报错：`Failed to connect to github.com port 443`**

原因：GitHub 在国内被 GFW 墙了（GFW = 防火长城，中国的网络审查系统）。

尝试镜像站 `ghproxy.net`，同样连不上。

**解决**：打开 VPN，但依然报 SSL 错误。最终关闭 Git 的 SSL 验证后成功：

```bash
git config --global http.sslVerify false
git clone ... themes/PaperMod
```

下载下来了，但主题目录里有一个 `.github` 文件夹，和我们项目自己需要的 `.github` 配置冲突——删掉它。

## 六、配置网站

编辑 `hugo.toml`：

```toml
baseURL = 'https://forest-sea.github.io/'
locale = 'zh-cn'
title = '我的博客'
theme = 'PaperMod'
```

解释：
- `baseURL` — 网站最终的上线地址
- `locale` — 语言，设为中文简体
- `theme` — 用哪个皮肤

还配了一堆 PaperMod 的参数：显示阅读时间、分享按钮、代码复制、顶部导航栏等。

## 七、创建第一篇文

在 `content/posts/` 下新建 `hello-world.md`：

```markdown
---
title: "第一篇博客：Hello World"
date: 2026-05-27
tags: ["随笔"]
summary: "这是我的第一篇博客文章。"
---

正文内容。
```

文件开头的 `---` 包起来的部分叫"元数据"（front matter），包含标题、日期、标签、摘要。

## 八、本地预览

```bash
hugo server -D --bind 0.0.0.0 --port 1313
```

浏览器打开 `http://localhost:1313`，能看到博客。

**注意**：`localhost` 只能本机访问。别的设备（手机）要访问得用局域网 IP，比如 `http://172.16.2.23:1313`。

## 九、文章页和搜索页 404——第二个坑

首页好好的，但点"文章"和"搜索"都 404。

原因：Hugo 的机制是——每个 URL 都需要对应的内容文件。归档页和搜索页需要专门创建。

**解决**：创建 `content/archives.md` 和 `content/search.md`，指定特殊的布局（`layout: archives` 和 `layout: search`），并在配置里加上 `mainSections = ["posts"]` 告诉归档页去哪找文章。

## 十、推送到 GitHub——第三个坑

安装 GitHub 命令行工具：

```bash
winget install GitHub.cli
```

登录时再次碰壁。`gh auth login` 打不开浏览器，因为连接被 GFW 拦了。

**解决**：在 GitHub 网页上手动生成 Personal Access Token（个人令牌），勾选 `repo` 和 `workflow` 权限，然后用 token 登录：

```bash
gh auth login --with-token
```

又报错：`missing required scope 'read:org'`。但这个权限不是必需的，用环境变量的方式绕过去：

```bash
export GITHUB_TOKEN="ghp_xxxx"
```

账户名：**Forest-Sea**。

## 十一、创建仓库并推送

```bash
git add .
git commit -m "初始化博客"
gh repo create Forest-Sea.github.io --public --push
```

报错：`no commits found`。因为还没执行 `git commit`。补上提交后，代码推上去了。

## 十二、设置 GitHub Actions 自动部署——第四个坑

创建 `.github/workflows/hugo.yaml` 工作流文件。这个文件告诉 GitHub：每次有人 push 代码，自动执行"安装 Hugo → 构建网站 → 部署到 Pages"。

第一次运行：构建成功，部署失败。部署任务的日志直接找不到。

原因：GitHub Pages 的 `environment` 有分支保护规则，只允许从 `master` 分支部署，但我们用的分支是 `main`。

换个思路，先用 `peaceiris/actions-gh-pages` 把构建结果推到 `gh-pages` 分支，再让 Pages 从那个分支读取：

```yaml
- name: Deploy
  uses: peaceiris/actions-gh-pages@v4
  with:
    publish_dir: ./public
    publish_branch: gh-pages
```

同时修改 Pages 设置，从 `gh-pages` 分支部署。

**成功。** 状态变成 `built`。

## 十三、手机访问不了——第五个坑

网站虽然部署成功了，但从中国手机访问 `https://forest-sea.github.io/` 打不开。

原因：`github.io` 域名在中国被 DNS 污染，即使开 VPN，手机 VPN 可能走的是分流模式，DNS 查询还是国内服务器，解析不出正确的 IP。

**解决方法**：
1. 装 Cloudflare WARP（搜 "1.1.1.1"）
2. 或者把 VPN 改成全局模式
3. 或者换 Cloudflare Pages 托管（国内访问更稳）

## 整个过程中遇到的错误总结

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `hugo: command not found` | 环境变量未刷新 | 用完整路径 |
| GitHub 连不上 | GFW 墙了 | 开 VPN + 关 SSL 验证 |
| 文章/搜索 404 | 缺少页面文件 | 创建 archives.md 和 search.md |
| gh auth 登录失败 | GFW 墙了 | 用 Personal Access Token |
| Pages 部署失败 | 分支保护规则 | 改用 gh-pages 分支部署 |
| 手机打不开 | DNS 污染 | Cloudflare WARP 或全局 VPN |

## 最终成果

不花一分钱，一个属于自己的个人博客上线了。

以后每写一篇新文章，只需要三步：

```bash
git add .
git commit -m "新文章"
git push
```

其他的一切自动化。

## 学到了什么

- **静态网站 vs 动态网站**：静态网站是一堆 HTML 文件，不需要服务器运行程序，免费托管就够了
- **Hugo**：全球最快的静态网站生成器
- **Git + GitHub**：版本控制和代码托管
- **GitHub Actions**：自动化流水线
- **GitHub Pages**：免费网页托管
- **GFW**：中国的网络长城，访问国外网站受限
- **DNS 污染**：域名解析被干扰，导致网站打不开
