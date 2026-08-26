---
title: GitHub Fork 项目的标准开发流程：同步上游、创建开发分支与提交 Pull Request
date: 2026-08-17 12:00:00
updated: 2026-08-17 12:00:00
description: 以实际 Fork 的 Halo 项目为例，完整记录 GitHub Fork 项目的标准开发流程：同步上游、创建开发分支、提交 Pull Request，并详解 Git 代理配置
keywords: [GitHub, Fork, Git, upstream, Pull Request, 开源贡献, 代理配置]
tags:
  - GitHub
  - Git
  - 开源
  - 协作
categories:
  - [技术, 工具]
cover: /img/bg3.png
top_img:

---

## 前言

在参与 GitHub 开源项目时，一个非常常见的工作流程是：

> Fork 原作者仓库 → Clone 自己的 Fork → 配置 upstream → 保持 main 与原作者同步 → 创建开发分支 → 修改代码 → Push 到自己的 GitHub → 向原作者提交 Pull Request。

本文以我实际 Fork 的 Halo 项目为例，完整记录这一流程，同时介绍 Git、GitHub、`origin`、`upstream`、本地分支、远程分支以及代理配置之间的关系。

---

## 一、Fork、Clone、origin 和 upstream 分别是什么？

假设原作者的仓库是：

```text
https://github.com/halo-dev/halo.git
```

我在 GitHub 上点击 Fork 后，会得到自己的仓库：

```text
https://github.com/bedhere/halo.git
```

然后将自己的 Fork 克隆到本地。

此时整个关系可以理解为：

```text
原作者 GitHub 仓库
halo-dev/halo
      │
      │ Fork
      ↓
自己的 GitHub 仓库
bedhere/halo
      │
      │ Clone
      ↓
本地项目目录
```

Git 默认会把你 Clone 的远程仓库命名为：

```text
origin
```

所以最开始执行：

```bash
git remote -v
```

可能只能看到：

```text
origin  https://github.com/bedhere/halo.git (fetch)
origin  https://github.com/bedhere/halo.git (push)
```

这里：

* `origin`：自己的 GitHub Fork
* `upstream`：原作者的 GitHub 仓库

但是 `upstream` 不会自动出现，需要自己配置。

---

## 二、添加原作者仓库为 upstream

进入本地项目目录：

```powershell
cd "E:\code project\helo"
```

添加原作者仓库：

```powershell
git remote add upstream https://github.com/halo-dev/halo.git
```

然后检查：

```powershell
git remote -v
```

正常情况下应该看到：

```text
origin    https://github.com/bedhere/halo.git (fetch)
origin    https://github.com/bedhere/halo.git (push)

upstream  https://github.com/halo-dev/halo.git (fetch)
upstream  https://github.com/halo-dev/halo.git (push)
```

至此，远程仓库关系就建立好了：

```text
upstream
   │
   └── halo-dev/halo
       原作者仓库

origin
   │
   └── bedhere/halo
       自己 Fork 的仓库
```

---

## 三、获取原作者最新代码

执行：

```powershell
git fetch upstream
```

这条命令的作用不是直接修改当前代码，而是：

> 从原作者仓库下载最新的分支、提交、Tag 等信息到本地 Git 数据库。

例如：

```text
upstream/main
upstream/release-2.26
upstream/master
...
```

都会被获取下来。

这里需要区分：

```text
main
```

和：

```text
upstream/main
```

它们不是同一个东西。

`main` 是本地分支，而：

```text
upstream/main
```

表示：

> 本地记录的"原作者远程 main 分支最新状态"。

---

## 四、让本地 main 与原作者完全一致

我的目标是：

> `main` 不进行个人开发，只用于跟踪原作者代码。

因此首先切换到：

```powershell
git switch main
```

然后执行：

```powershell
git reset --hard upstream/main
```

这条命令意味着：

> 强制让当前本地 `main` 与 `upstream/main` 完全相同。

此时关系变成：

```text
halo-dev/halo
upstream/main
      │
      ↓
本地 main
```

### 注意

`git reset --hard` 会丢弃当前分支尚未保存的修改。

因此执行之前最好先检查：

```powershell
git status
```

如果有自己需要保留的代码，不应该直接执行 `reset --hard`。

---

## 五、将自己的 GitHub main 同步为原作者版本

本地 `main` 同步之后，还需要更新自己的 GitHub Fork：

```powershell
git push origin main --force-with-lease
```

于是：

```text
原作者
halo-dev/halo:main
        │
        ↓
本地 main
        │
        ↓
自己的 GitHub
bedhere/halo:main
```

三者保持一致。

这里使用：

```text
--force-with-lease
```

而不是直接：

```text
--force
```

原因是 `--force-with-lease` 相对更加安全。

如果远程分支发生了自己不知道的新变化，它通常不会直接粗暴覆盖。

---

## 六、为什么不要直接在 main 上开发？

如果直接在 `main` 上写自己的功能，后面原作者继续更新时，会出现：

```text
原作者 main
      │
      ├── 原作者的新提交
      │
你的 main
      └── 自己的修改
```

久而久之，`main` 会越来越难同步。

更合理的设计是：

```text
main
│
│ 只负责跟踪原作者
│
└── dev
    │
    └── 自己开发
```

所以：

> `main` 保持干净，`dev` 用于自己的开发。

---

## 七、创建本地 dev 开发分支

确保当前 `main` 已经是最新状态，然后创建：

```powershell
git switch -c dev
```

这里：

```text
-c
```

表示：

> 创建一个新分支，并立即切换过去。

检查：

```powershell
git branch
```

此时应该显示：

```text
* dev
  main
```

`*` 表示当前所在分支。

所以：

```text
* dev
```

说明当前正在 `dev` 上开发。

---

## 八、为什么 GitHub 网页上仍然只有 main？

这是一个刚开始学习 Git 时非常容易困惑的问题。

执行：

```powershell
git switch -c dev
```

只是创建了：

> 本地 dev 分支

这并不会自动在 GitHub 上创建分支。

所以此时：

```text
本地：
main
dev

GitHub：
main
```

完全正常。

GitHub 页面显示的是远程仓库分支，而：

```bash
git branch
```

默认显示的是本地分支。

---

## 九、将本地 dev 推送到自己的 GitHub

执行：

```powershell
git push -u origin dev
```

成功后通常会看到：

```text
[new branch] dev -> dev

branch 'dev' set up to track 'origin/dev'
```

此时：

```text
本地 dev
    │
    │ git push
    ↓
origin/dev
    │
    ↓
bedhere/halo:dev
```

再刷新 GitHub 页面，就可以看到：

```text
main
dev
```

两个分支。

---

## 十、-u 的作用是什么？

第一次执行：

```powershell
git push -u origin dev
```

其中：

```text
-u
```

实际上是：

```text
--set-upstream
```

它告诉 Git：

> 以后本地 `dev` 默认跟踪 `origin/dev`。

建立跟踪关系之后，以后就不必每次都写：

```powershell
git push origin dev
```

而可以直接：

```powershell
git push
```

同样：

```powershell
git pull
```

也知道应该从哪个远程分支获取代码。

---

## 十一、日常开发流程

以后自己的代码都在：

```text
dev
```

分支上修改。

先确认所在分支：

```powershell
git branch
```

如果不在 `dev`：

```powershell
git switch dev
```

修改代码之后检查：

```powershell
git status
```

然后：

```powershell
git add .
```

提交：

```powershell
git commit -m "feat: add xxx feature"
```

上传：

```powershell
git push
```

整个流程就是：

```text
修改代码
   ↓
git add .
   ↓
git commit
   ↓
本地 dev 产生新的 Commit
   ↓
git push
   ↓
origin/dev
```

---

## 十二、原作者更新后，如何同步？

这是 Fork 项目中最重要的日常操作之一。

假设原作者继续提交了很多代码：

```text
halo-dev/halo:main
```

发生变化。

首先切换到自己的 `main`：

```powershell
git switch main
```

获取原作者最新版：

```powershell
git fetch upstream
```

强制同步：

```powershell
git reset --hard upstream/main
```

然后更新自己的 GitHub Fork：

```powershell
git push origin main --force-with-lease
```

此时：

```text
upstream/main
      =
本地 main
      =
origin/main
```

然后回到自己的开发分支：

```powershell
git switch dev
```

把最新 `main` 合入：

```powershell
git merge main
```

最后：

```powershell
git push
```

所以以后同步上游代码时，可以记住这一整套：

```powershell
git switch main

git fetch upstream

git reset --hard upstream/main

git push origin main --force-with-lease

git switch dev

git merge main

git push
```

结构如下：

```text
halo-dev/halo
upstream/main
      │
      │ fetch + reset
      ↓
本地 main
      │
      │ merge
      ↓
本地 dev
      │
      │ push
      ↓
origin/dev
```

---

## 十三、如何向原作者提交 Pull Request？

当自己的功能开发完成以后，需要让原作者决定是否将修改合并进官方仓库。

这就是：

```text
Pull Request
```

简称：

```text
PR
```

需要注意：

> Pull Request 是 GitHub 提供的功能，并不是 Git 本身的命令。

Git 本身主要负责：

```text
commit
branch
merge
fetch
pull
push
```

GitHub 在 Git 之上增加了：

```text
Issue
Pull Request
Code Review
Actions
...
```

---

## 十四、PR 的方向一定要看清楚

假设我要把自己的：

```text
bedhere/halo:dev
```

提交给官方：

```text
halo-dev/halo:main
```

那么 GitHub 中应该选择：

```text
base repository:
halo-dev/halo

base:
main
```

以及：

```text
head repository:
bedhere/halo

compare:
dev
```

关系就是：

```text
bedhere/halo:dev
        │
        │ Pull Request
        ↓
halo-dev/halo:main
```

也就是：

> 请求原作者把我的 dev 修改合并到官方 main。

---

## 十五、更规范的方式：不要长期用 dev 直接提 PR

虽然：

```text
dev → upstream/main
```

技术上可以创建 PR，但实际参与大型开源项目时，更推荐：

> 一个功能 / 一个 Bug / 一个 PR / 一个分支。

例如我要修复登录问题。

首先保证 `main` 最新：

```powershell
git switch main
git fetch upstream
git reset --hard upstream/main
```

然后从最新 `main` 创建专门的分支：

```powershell
git switch -c fix/login-error
```

修改完成后：

```powershell
git add .
git commit -m "fix: resolve login error"
```

推送：

```powershell
git push -u origin fix/login-error
```

然后提交：

```text
bedhere/halo:fix/login-error
              │
              │ Pull Request
              ↓
halo-dev/halo:main
```

这种方式最大的优点是：

> 一个 PR 中不会混入其他尚未完成的代码。

所以更加推荐这样的分支结构：

```text
main
│
├── dev
│   └── 长期个人开发
│
├── fix/login-error
│   └── 一个 Bug 修复
│
├── feat/new-editor
│   └── 一个新功能
│
└── docs/update-readme
    └── 一次文档修改
```

---

## 十六、Git 无法连接 GitHub：代理问题

在实际操作中，我遇到了这样的错误：

```text
fatal: unable to access 'https://github.com/bedhere/halo.git/':
Recv failure: Connection was reset
```

以及：

```text
Failed to connect to github.com port 443
```

浏览器虽然可以打开 GitHub，但 Git 命令不一定会自动使用系统或者代理软件的代理。

我的本地代理地址为：

```text
127.0.0.1:7892
```

最终使用 SOCKS5 代理解决。

设置 Git 全局代理：

```powershell
git config --global http.proxy socks5h://127.0.0.1:7892
```

以及：

```powershell
git config --global https.proxy socks5h://127.0.0.1:7892
```

检查：

```powershell
git config --global --get http.proxy
```

得到：

```text
socks5h://127.0.0.1:7892
```

说明配置已经生效。

之后再次执行：

```powershell
git push -u origin dev
```

成功：

```text
[new branch] dev -> dev

branch 'dev' set up to track 'origin/dev'
```

---

## 十七、这个操作叫什么？

执行：

```powershell
git config --global http.proxy ...
git config --global https.proxy ...
```

通常称为：

> 给 Git 配置全局 HTTP/HTTPS 代理。

或者：

> 配置 Git Global Proxy。

其中：

```text
git config
```

表示修改 Git 配置。

```text
--global
```

表示：

> 对当前 Windows 用户下所有 Git 仓库生效。

因此它不是：

```text
Windows 系统代理
```

也不是：

```text
GitHub 仓库配置
```

而是：

```text
Git 自己的全局网络代理配置
```

---

## 十八、如何取消 Git 全局代理？

如果以后代理软件关闭，而 Git 仍然配置：

```text
127.0.0.1:7892
```

那么 Git 可能会报：

```text
Failed to connect to 127.0.0.1 port 7892
```

此时可以删除：

```powershell
git config --global --unset http.proxy
```

以及：

```powershell
git config --global --unset https.proxy
```

检查：

```powershell
git config --global --get http.proxy
git config --global --get https.proxy
```

没有输出，就说明代理配置已经删除。

查看所有代理相关配置：

```powershell
git config --global --get-regexp proxy
```

---

## 十九、HTTP 代理与 SOCKS5 代理

如果代理软件提供的是 HTTP 代理端口，可以配置：

```powershell
git config --global http.proxy http://127.0.0.1:7892
git config --global https.proxy http://127.0.0.1:7892
```

如果代理端口是 SOCKS5：

```powershell
git config --global http.proxy socks5h://127.0.0.1:7892
git config --global https.proxy socks5h://127.0.0.1:7892
```

我最终使用的是：

```text
socks5h://127.0.0.1:7892
```

并成功完成 GitHub Push。

---

## 二十、如何查看当前 Git 状态？

下面几个命令在日常使用中非常重要。

### 查看当前分支

```powershell
git branch
```

例如：

```text
* dev
  main
```

### 查看本地和远程所有分支

```powershell
git branch -a
```

可能看到：

```text
* dev
  main
  remotes/origin/dev
  remotes/origin/main
  remotes/upstream/main
```

### 查看分支跟踪关系

```powershell
git branch -vv
```

例如：

```text
* dev   a559d155a [origin/dev] ...
  main  a559d155a [origin/main] ...
```

### 查看远程仓库

```powershell
git remote -v
```

### 查看工作区状态

```powershell
git status
```

---

## 二十一、最终形成的 Git 工作流

经过配置以后，我的仓库结构可以理解为：

```text
                        GitHub
                  halo-dev/halo
                  upstream/main
                        │
                        │ fetch
                        ↓
                  ┌───────────┐
                  │ 本地 main │
                  └───────────┘
                        │
             ┌──────────┴──────────┐
             │                     │
             ↓                     ↓
      origin/main                 dev
   bedhere/halo:main               │
                                   │ 开发
                                   ↓
                              本地 Commit
                                   │
                                   │ push
                                   ↓
                              origin/dev
                                   │
                                   │ Pull Request
                                   ↓
                           upstream/main
```

其中最重要的原则是：

> **main 负责跟踪原作者，开发工作不要直接放在 main。**

---

## 二十二、日常最常使用的命令

如果只记一套流程，可以记下面这些。

### 自己开发

```powershell
git switch dev

git status

git add .

git commit -m "feat: xxx"

git push
```

### 同步原作者最新版

```powershell
git switch main

git fetch upstream

git reset --hard upstream/main

git push origin main --force-with-lease

git switch dev

git merge main

git push
```

### 创建一个用于官方 PR 的独立分支

```powershell
git switch main

git fetch upstream

git reset --hard upstream/main

git switch -c fix/xxx
```

修改代码后：

```powershell
git add .

git commit -m "fix: xxx"

git push -u origin fix/xxx
```

然后：

```text
自己的 fix/xxx
        ↓
   Pull Request
        ↓
原作者 main
```

---

## 二十三、一句话理解 origin 和 upstream

最后可以用一句话记忆：

```text
origin = 我的 GitHub
upstream = 原作者的 GitHub
```

再加一句：

```text
main = 跟作者
dev / feature / fix = 写自己的代码
```

于是整个 Fork 开发流程就非常清晰了：

```text
Fork
 ↓
Clone
 ↓
添加 upstream
 ↓
同步 upstream/main
 ↓
保持自己的 main 干净
 ↓
创建开发分支
 ↓
修改 + commit
 ↓
push 到 origin
 ↓
Pull Request
 ↓
原作者 Review
 ↓
Merge
```

这就是一套比较规范、适合长期参与 GitHub 开源项目的 Fork 开发工作流。
