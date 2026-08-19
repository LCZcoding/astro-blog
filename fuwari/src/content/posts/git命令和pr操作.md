---
title: git命令和pr操作
published: 2026-08-19
description: '笔记：关于常用的git命令和pr操作'
image: ''
tags: [to3, git, pr, 笔记]
category: '笔记'
draft: false 
lang: ''
---
本人写的 [带图飞书文档](https://rcnujngjny8w.feishu.cn/wiki/JaCZweUVBixZ3DkVMmHc2m9Fn1c?from=from_copylink)、更加详细具体的可以参考[菜鸟](https://www.runoob.com/git/git-tutorial.html)

# 克隆远程仓库到本地

```bash
git clone <仓库地址https或ssh（需要key）>
```

# 展示当前仓库对应的远程仓库(展示推拉目的仓库链接)

```bash
git remote -v
```

# 展示当前分支名字

```bash
git branch --show-current
```

# 创建新分支（注意分支命名规范）

```bash
git checkout -b feature-login
```
 
# 展示本地文件跟踪、暂存情况

```bash
git status
```
:::note
输出：</br>
未跟踪（Untracked）： 新创建的文件最初是未跟踪的。它们存在于工作目录中，但没有被 Git 跟踪。</br>
已跟踪（Tracked）： 通过 git add 命令将未跟踪的文件添加到暂存区后，文件变为已跟踪状态。</br>
已修改（Modified）： 对已跟踪的文件进行更改后，这些更改会显示为已修改状态，但这些更改还未添加到暂存区。</br>
已暂存（Staged）： 使用 git add 命令将修改过的文件添加到暂存区后，文件进入已暂存状态，等待提交。</br>
已提交（Committed）： 使用 git commit 命令将暂存区的更改提交到本地仓库后，这些更改被记录下来，文件状态返回为已跟踪状态。</br>
被 .gitignore 匹配的文件，即使存在于工作区，也不会显示为 Untracked。
:::

# 解释git的4个区域

1. 工作区  (Working Directory) 
实际修改文件的地方。在这里写代码、删删改改。

2. 暂存区 (Staging Area) 
添加到暂存区指令：
```bash
git add <filename or .> #意义：确认准备提交到本地仓库
```

3. 本地仓库 (Local Repository)
提交到本地仓库指令：
```bash
git commit -m "feat: 添加登录功能的基础框架" # 意义：改动正式成为了项目历史的一部分。即便你之后改乱了，也可以随时从这里找回。
```

4. 远程仓库 (Remote)
在推送本地更改之前，最好从远程仓库拉取最新的更改，以避免冲突：
```bash
git pull origin <main or feature-login>
```

提交指令：
```bash
git push -u origin feature-login #（第一次提交新分支要用 -u 创建） 意义：备份代码，并与团队共享进度。
```

# 合并更改
## 1. 切换到目标分支（接收方）

```bash
git checkout main
```


## 2. 确保目标分支是最新的（避免不必要的冲突）

```bash
git pull origin main
```

## 3. 执行合并命令

```bash
git merge feature-login
```

# 解决冲突

当合并过程中出现冲突时，Git 会标记冲突文件，你需要手动解决冲突。打开冲突文件，按照标记解决冲突。
标记冲突解决完成后暂存提交：
```bash
git add <conflict-file-name>
git commit
```

# 删除分支

```bash
git branch -d <branch name>
```

# 基础操作(核心汇总版)

```bash
git reset # 撤销提交并取消暂存，改动还在工作区 回退到本地仓库的上一个版本
git rm filename.txt # 将文件从暂存区和工作区删除
git mv old_name.js new_name.js # 移动或重命名工作区文件
git checkout <branch-name> # 分支切换
git log # 查看提交记录
git pull # 下载远程代码并合并
git push # 推送并合并
```


# 分支管理

```bash
git checkout -b <branchname> # 创建并切换分支，没有-b 是切换
git branch # 查看所有分支
git branch -r # 查看远程分支
git branch -a # 查看所有本地和远程分支
git merge <branchname> # 将其他分支合并到当前分支
```


# github贡献操作 pr（省略了常规git操作，包括分支等）

## 先fork下来，再跟踪上游仓库（原作者仓库）upstream：
```bash
git remote add upstream https://github.com/agegr/pi-web.git 
#
```

## （不要忘了新建分支）完成工作后对无关改动还原：

```bash
git checkout upstream/main -- package-lock.json
# 解释：从远程追踪分支 upstream/main 中，强制恢复（或覆盖）当前工作区中的 package-lock.json 文件。
```

## review+跑测试

两套测试：

```bash
node --experimental-strip-types --test lib/request-security.test.mjs 
npm test #（一般是作者设置的，看具体情况）
#命令分析
# 1. node --experimental-strip-types --test lib/request-security.test.mjs
# --experimental-strip-types: Node.js 的实验性标志，允许直接运行包含 TypeScript 类型注解的 .mjs 文件（无需编译）
# --test: 启用 Node.js 内置的测试运行器
# lib/request-security.test.mjs: 具体的测试文件
# 作用：直接使用 Node.js 原生测试框架运行一个特定的安全相关测试文件。
# 2. npm test
# 这是 npm 的标准脚本命令，会执行 package.json 中定义的 test 脚本。
# 作用：运行项目配置的完整测试套件。
```

## pr提交和issue提交：提交issue注意模版。到compare界面再提交pr

compare url：
```bash
https://github.com/agegr/pi-web/compare/main...LCZcoding:fix/chromium-origin-port-stripping
# /agegr/pi-web/ 是原作者仓库位置，后面的是提交用户名字和提交内容标题
```
