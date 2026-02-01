---
title: git托管代码
date: 2026-01-22 03:12:00
updated: 2026-01-22 03:12:00   # 更新日期（可选）
description: 详解如何把 Hexo 源码与静态页双仓库托管到 GitHub，实现自动化部署
keywords: [Hexo, GitHub, Butterfly, 双仓库]
tags:
  - Hexo
  - Git
categories:
  - [技术, 博客]
cover: 
top_img: 
---

## 总结

把 Hexo 的「源码」和「生成后的静态页面」分别放进两个 Git 仓库：
- 一个仓库专门给 **GitHub Pages** 用来对外访问
- 一个仓库只负责 **源码备份与回滚**

这样做的结果是：**发布不影响备份，备份不干扰发布，部署流程极其干净。**

---

## 1. 最终结构与效果

### 仓库结构

| 仓库名 | 作用 | 访问地址 |
|------|------|--------|
| `bedhere.github.io` | 存放 Hexo 生成的静态页面（public） | https://bedhere.github.io |
| `hexo-blog-source` | Hexo 源码仓库（文章 / 主题 / 配置） | https://github.com/bedhere/hexo-blog-source |

### 日常使用体验

- 本地 **只有一个 Hexo 项目目录**
- Git 配置 **两个 remote**
- 写作发布流程高度固定：

```
hexo d          # 发布博客
git push        # 顺手备份源码
```
##  前期准备

### 2.1 环境要求

- ✅ 已安装 Node.js
- ✅ 已安装 Hexo
- ✅ Hexo 项目可正常 `hexo s` 运行
- ✅ 已配置主题（如 Butterfly）

---

### 2.2 仓库准备

在 GitHub 上新建 **两个空仓库**（❌ 不要勾选 README）：

####  bedhere.github.io

- **用途**：存放 `public/` 静态文件（GitHub Pages）

####  hexo-blog-source

- **用途**：存放 Hexo 源码
- **作用**：便于回滚、迁移、备份

---

## 3️⃣ 整体流程概览

完整配置流程如下：

1. 初始化本地 Git 仓库
2. 添加两个远程仓库
3. 配置 Hexo Git 自动部署
4. 首次提交并发布博客
5. 汇总常见问题与解决方案

---

## 4️⃣ 详细配置步骤

### 4.1 初始化 Git（仅首次）

进入 Hexo 根目录并初始化 Git：
```
cd E:\BlogFile   # 替换为你的 Hexo 根目录
git init
git checkout -b main
```
---

### 4.2 配置 .gitignore

防止无关文件被提交到源码仓库：
```
node_modules/
public/
.deploy_git/
db.json
*.log
```
---

### 4.3 添加双远程仓库
```
git remote add pages  https://github.com/bedhere/bedhere.github.io.git
git remote add source https://github.com/bedhere/hexo-blog-source.git
```
**说明：**

- pages：专用于 Hexo 静态页面部署
- source：专用于源码备份

---

### 4.4 安装并配置 Hexo 部署插件

安装 Hexo 部署插件：
```
npm install hexo-deployer-git --save
```
在 _config.yml 文件末尾添加：
```
deploy:
type: git
repo: pages
branch: main
message: "site updated: {{ now('YYYY-MM-DD HH:mm:ss') }}"
```
---

### 4.5 首次提交与发布

#### 4.5.1 提交源码
```
git add .
git commit -m "init: hexo source backup"
git push -u source main
```
#### 4.5.2 生成并部署静态页
```
hexo clean && hexo d
```
## 5️⃣ 日常写作流程

### 5.1 新建文章并本地预览
```
hexo new post "文章标题"
hexo s
```
---

### 5.2 发布博客并备份源码
```
hexo clean && hexo d
git add .
git commit -m "post: xxx"
git push source
```
---

## 6️⃣ 常见问题与解决方案

### 6.1 Git 推送失败

**问题原因：**

- HTTPS 443 端口不稳定
- 代理工具与 Git 冲突

✅ **推荐方案：使用 SSH**
```
git remote set-url pages git@github.com:bedhere/bedhere.github.io.git
git remote set-url source git@github.com:bedhere/hexo-blog-source.git
```


