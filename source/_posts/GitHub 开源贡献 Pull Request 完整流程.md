---
title: GitHub 开源贡献 Pull Request 完整流程
date: 2026-07-22 10:00:00
updated: 2026-07-22 10:00:00
description: 从 Fork 到 Merge，详解给 GitHub 开源项目提 PR 的完整流程与最佳实践，涵盖分支管理、Commit 规范、Code Review 应对策略
keywords: [GitHub, Pull Request, 开源贡献, Git, Fork, Code Review, 协作开发]
tags:
  - GitHub
  - Git
  - 开源
  - 协作
categories:
  - [技术, 工具]
cover: /img/food2.png
top_img:

---

## 前言

给开源项目提 Pull Request 是参与社区协作最直接的途径。但第一次走完这套流程，中间确实有不少容易卡住的细节——从 Fork 仓库到最终合入，每个环节都有约定俗成的做法。

本文以一次实际 PR 提交为蓝本，把完整流程拆成十个步骤，末尾整理了几条通用原则。

---

## 一、整体流程

```
选项目 → Fork → 克隆到本地 → 创建功能分支 → 本地开发修改
→ 本地验证测试 → 提交代码 → Push 到个人仓库
→ 创建 Pull Request → Code Review → 合入
```

三个最容易出问题的环节：分支命名、Commit 规范、本地验证。后面会逐一展开。

---

## 二、详细步骤

### 1. 选择开源项目

动手前先读几个文件：

| 文件 | 内容 |
|------|------|
| `README.md` | 项目定位、技术栈、快速启动 |
| `CONTRIBUTING.md` | 贡献规范，必须逐条看完 |
| `CODE_OF_CONDUCT.md` | 社区行为准则 |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR 模板 |

CONTRIBUTING.md 是重点。关注以下几点：

- **Commit 规范**：很多项目要求 Conventional Commits（`feat:`、`fix:` 等前缀）
- **分支命名**：是否有约定
- **本地检查命令**：lint、format、test 的命令是什么
- **合入方式**：squash merge、rebase merge 还是普通 merge

如果项目没有 CONTRIBUTING.md，可以参考同类项目的惯例，或者先提 Issue 询问。

---

### 2. Fork 项目仓库

在 GitHub 上打开原项目页面，点击右上角的 **Fork**，将项目复制到自己的 GitHub 账号下。

```
原项目：github.com/作者/项目名
个人：  github.com/<你的用户名>/项目名
```

Fork 时不需要修改选项，直接确认即可。

---

### 3. 克隆到本地

克隆自己 Fork 的仓库（注意不是原项目）：

```bash
git clone git@github.com:<你的用户名>/项目名.git
cd 项目名
```

根据 README 安装依赖。

建议将原项目仓库添加为 upstream 远程，便于后续同步更新：

```bash
git remote add upstream git@github.com:作者/项目名.git
git remote -v
# origin   → 你的 Fork（有推送权限）
# upstream → 原项目（只读）
```

---

### 4. 创建功能分支

不要在 main 或 master 分支上直接开发。

```bash
git checkout main
git pull upstream main
git checkout -b feat/你的功能名
```

分支命名没有强制约定，但建议与 commit type 对应：

| 场景 | 分支名示例 |
|------|-----------|
| 新功能 | `feat/keyboard-nav` |
| Bug 修复 | `fix/login-crash` |
| 重构 | `refactor/api-client` |
| 文档 | `docs/api-usage` |
| 杂项 | `chore/update-deps` |

PR 标题默认会使用分支名，命名清晰一些能帮维护者快速判断 PR 类型。

---

### 5. 本地开发修改

这个阶段最花时间。三个建议：

**先读懂再动手。** 把涉及的文件通读一遍，理清模块之间的依赖关系和数据流。改动涉及多个文件的话，先列一个改动清单。

**控制改动范围。** 如果改动涉及多个不相关的模块，考虑拆成多个 PR。每个 PR 只做一件事，review 和回溯都更方便。

**保持代码风格一致。** 不改变原文件的缩进方式（空格还是 Tab），不重命名已有的变量或函数（除非重构任务就是做这个），不修改与本次改动无关的代码。

---

### 6. 本地验证测试

提交前必须跑通项目要求的全部检查。不要等到 CI 才发现问题。

通常包含三类检查：

```bash
# 1. 代码格式检查
npx prettier --write .
# 或
cargo fmt

# 2. Lint 检查
npx eslint .
# 或
cargo clippy -- -D warnings

# 3. 单元测试
npm test
# 或
cargo test
```

关于 CI 有一点需要注意：**CI 检查的是 PR 分支与目标分支合并后的代码，而非 PR 分支的原始提交。** 也就是说即使本地全过了，如果合并时有冲突或其他人先合入了代码改动同一区域，CI 依然可能失败。

---

### 7. 提交代码

Commit message 是协作的"最小文档"。一条好的 message 应该让别人不点开 diff 也能明白变更意图。

推荐使用 Conventional Commits 格式：

```
<type>: 简短描述
```

常用 type：

| Type | 使用场景 | 示例 |
|------|---------|------|
| `feat` | 新功能 | `feat: 添加键盘导航支持` |
| `fix` | Bug 修复 | `fix: 修复登录页白屏问题` |
| `refactor` | 重构 | `refactor: 提取公共认证逻辑` |
| `docs` | 文档变更 | `docs: 更新 API 使用说明` |
| `style` | 格式调整 | `style: 排序 import 语句` |
| `test` | 测试相关 | `test: 添加用户注册测试用例` |
| `chore` | 构建/CI/依赖 | `chore: 升级 TypeScript 到 5.x` |

提交时只添加改动过的文件：

```bash
git add src/components/Button.tsx src/utils/helpers.ts
git commit -m "feat: 添加键盘快捷键导航支持"
```

---

### 8. Push 到个人仓库

```bash
git push origin feat/你的分支名
```

第一次推送后，GitHub 会返回一个快捷链接：

```
remote: Create a pull request for 'feat/xxx' on GitHub by visiting:
remote: https://github.com/<你的用户名>/项目名/pull/new/feat/xxx
```

推送后发现还有遗漏，继续 commit 再 push 即可。分支名不变的话 PR 会自动更新，不需要重新创建。

---

### 9. 创建 Pull Request

在 Fork 的仓库页面会出现 **Compare & pull request** 按钮。

确认页面上的几个关键字段：

| 字段 | 说明 |
|------|------|
| Base repository | 原项目仓库 |
| Base branch | 原项目的 main（或目标分支） |
| Head repository | 你的 Fork |
| Compare branch | 你的功能分支 |

PR 描述按模板填写。没有模板的话，至少包含以下内容：

```
## 改动内容

概括改了什么、为什么改。

## 测试方法

1. 如何验证改动前的表现
2. 如何验证改动后的效果
3. 附截图或录屏更佳

## 检查清单

- [ ] 格式检查通过
- [ ] Lint 检查通过
- [ ] 测试全部通过
```

PR 标题同样用 Conventional Commits 格式，如 `feat: 添加键盘导航支持`。

---

### 10. Code Review

提交后维护者会审查代码。结果通常有三种：

| 结果 | 含义 | 应对方式 |
|------|------|---------|
| Approved | 审核通过 | 等待维护者合入，或有权限则自行 squash and merge |
| Request Changes | 需要修改 | 本地修复后重新 commit 并 push，PR 自动更新 |
| Closed | PR 被关闭 | 查看关闭理由，如有疑问可以在 Discussion 中讨论 |

关于 Review 的几点经验：

- **及时回应。** 拖太久，维护者可能已经忘了上下文。
- **每条评论都回应。** 改好了就 resolve，觉得不需要改就说明理由。
- **态度谦和。** 维护者基于对项目的长期维护经验给出建议，即使不认同，也先表达理解再展开讨论。

关于合入方式：很多项目使用 **Squash merge**，将 PR 中所有 commit 压缩为一条再合入。这意味着过程中可以随时 commit 保存进度，最终都会被压成一条干净的 commit。不过 review 时维护者还是会逐条看，所以 commit 内容也不能太随意。

---

## 三、关键原则

### 原则一：每个 PR 只做一件事

一个 PR 同时修 Bug、加功能、升依赖，reviewer 难以判断每个改动的意图，合入后出问题也不好回溯。

```
推荐做法：
PR #1: feat: 添加键盘快捷键导航
PR #2: fix: 修复深色模式文字颜色
PR #3: chore: 升级 ESLint 配置

不推荐：
PR #1: 修了几个 bug 并加了新功能
```

### 原则二：分支名与 commit type 保持一致

```
分支名   → feat/keyboard-navigation
Commit  → feat: 添加键盘导航支持
PR 标题 → feat: 添加键盘导航支持
```

三者 type 统一，维护者通过分支名就能对 PR 类型有一个大致判断。

### 原则三：本地检查全部通过后再 push

CI 队列可能需要排队，反复推送浪费资源也消耗维护者耐心。`git push` 之前，确保格式检查、lint、测试、构建全部通过。

### 原则四：PR 描述让维护者不看 diff 也能理解

一个好的 PR 描述回答三个问题：

1. **改了什么？**—— 一句话概括
2. **为什么改？**—— 关联 Issue 或问题描述
3. **怎么验证？**—— 测试步骤或截图

### 原则五：尊重项目规范

先读 CONTRIBUTING.md 再动手。如果有 Discussion 或 Issue 讨论区，先确认思路再写代码，避免做无用功。遵守项目的 commit 格式、代码风格和合入流程。

---

## 四、常见问题

### 4.1 提交后发现遗漏

```bash
git add 遗漏的文件
git commit --amend --no-edit      # 合并到上一个 commit
git push origin 分支名 --force-with-lease
```

`--force-with-lease` 比 `--force` 更安全，会检查远程分支是否有他人新推的提交，防止误覆盖。

### 4.2 原项目更新导致冲突

```bash
git checkout main
git pull upstream main
git checkout 你的分支
git merge main
# 解决冲突
git add .
git commit -m "merge: 解决与 upstream main 的冲突"
git push origin 你的分支
```

### 4.3 PR 被关闭

先确认关闭原因：

- **已在其他 PR 中修复**：可以关闭
- **项目方向不符**：先提 Discussion 确认思路再提交
- **代码质量不过关**：根据反馈改好后重新开 PR

---

## 五、总结

开源 PR 的流程看起来步骤多，但跑通一次之后大部分操作就是重复了。真正决定 PR 质量的是改动本身的设计和对项目规范的遵守。

```
Fork → 创建分支 → 开发 → 本地验证 → 提交 → Push → PR → Review → 合入
```

分支命名、commit 格式、本地验证——这三个环节多花十分钟，能给后续省下不少时间。

```bash
# 一套完整的 PR 工作流
git checkout main
git pull upstream main
git checkout -b feat/my-feature
# 开发修改 → 本地验证（lint + format + test）
git add <改动的文件>
git commit -m "feat: 一句话描述"
git push origin feat/my-feature
# 在 GitHub 上创建 PR
```

---

> **参考**：一次实际的 PR 提交经历  
> **核心知识点**：Fork 工作流 | 分支管理 | Conventional Commits | Squash Merge | Code Review  
> **适用场景**：向 GitHub 上的开源项目提交 Pull Request
