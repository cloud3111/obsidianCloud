---
title: Git总结
tags:
  - Git
categories: 编程
date: 2024-11-25 15:35:24
---

# Git 常用命令总结

## Git 基础概念
- Git 是一种分布式版本控制系统
- **工作区** → **暂存区** → **本地仓库** → **本地远程仓库** → **远程仓库**

| 状态               | 所在区域 | 含义                   | 如何产生             | 下一步操作                                   |
| ---------------- | ---- | -------------------- | ---------------- | --------------------------------------- |
| Untracked（未跟踪）   | 工作区  | 没提交过, 是新建的未加入暂存区     | 新建文件但没 `git add` | `git add` → Staged；或 `.gitignore` 忽略    |
| Unstaged（已跟踪未暂存） | 工作区  | 文件曾被提交过，但修改后还没保存到暂存区 | 修改已提交/已跟踪的文件     | `git add` → Staged；或 `git restore` 撤销修改 |
| Staged（已暂存）      | 暂存区  | 文件的当前修改已被保存到暂存区      | `git add` 已修改文件  | `git commit` → Committed                |
| Committed（已提交）   | 本地仓库 | 文件已提交至本地             | `git commit`     | `git push` → 远程仓库                       |
![17443477043361744347703409.png|700x335](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17443477043361744347703409.png)

```sh
git init # 将目录变成本地仓库
git add .
git commit -m 'xxx' # 提交到本地仓库
git remote add origin https://github.com/Tyson0314/profile # 关联远程仓库
git branch --set-upstream-to=origin/master master  # 本地分支关联远程分支
git pull origin master --allow-unrelated-histories # 允许合并不相关的历史
git push -u origin master  # 如果当前分支与多个主机存在追踪关系，则-u会指定一个默认主机，这样后面就可以不加任何参数使用git push。
```
## Git 基本操作

### 配置信息
```bash
#查看当前配置的git用户信息
git config –-list    

# 配置git提交用户名
git config --global user.name XXX    

# 配置git提交邮箱
git config --global user.email XXX            
```
### 创建本地仓库
```bash
# 创建文件夹
mkdir repository

# 进入文件夹
cd repository

# 初始化本地仓库,当前目录为git仓库 也就是repository
git init

# 将默认的 master 分支改名为 main
git branch -m main
```
### 克隆远程仓库
```bash
# 查看远程仓库
git remote -v

# 克隆远程仓库到本地 若使用http形式的需要配置 github账号密码, 推荐使用ssh
git clone https://github.com/cloud3111/cloud3111.git

# 添加远程仓库（若非克隆仓库）
git remote add origin https://github.com/cloud3111/cloud3111.git
```
---
## Git 提交与推送
### 创建提交文件
```bash
# 查看仓库状态
git status

# 写入并创建文件
echo "hello world" >> README.md

# 将文件添加到暂存区,在idea中add操作和commit操作是一起的
git add README.md

# 提交到本地仓库
git commit -m "详细信息"
```
### 查看日志
```bash
# 查看提交记录的简略信息  ---> 提交树
git log --online --graph

# 查看提交信息
git show <commit-id>

# 查看分支
git show <branch-name>

# 查看某次提交的文件内容
git show <commit-id>:<file-path>
```
### 删除文件
```bash
echo "rm test" > rm.txt

# 添加文件到暂存区并提交
git add rm.txt
git commit -m "version2"

# 删除文件 
# 在暂存区（index）中把文件标记为删除 加一个后缀 --cached 逻辑删除 当git push时远程仓库就会真正删除这个文件
# 同时也会在工作区（working directory）中物理删除文件
# git rm 是完全删除，加 --cached 就是只删除暂存区，保留本地文件
git rm rm.txt
git commit -m "version3"
```
### 推送与拉取
```bash
# 推送本地 main 分支到远程仓库 main 分支
git push -u origin main:main

# 从远程仓库拉取 main 分支到本地 main 分支
git pull origin main:main
```
---
## Git 分支管理
```bash
# 列出所有分支（包括远程分支）
git branch -a

# 创建并切换到 dev 分支 3种方式
git branch dev
git switch dev
git checkout -b dev
```
### 合并分支
```bash
# 合并 dev 分支到当前main分支
git merge dev
```
### 解决冲突
```bash
# 查看冲突文件内容 (工作区和暂存区之间的差异)
git diff  

# 想看暂存区和最新一次提交（HEAD）之间的差异
git diff --cached

# 想看当前工作区和最新一次提交（HEAD）之间的所有差异
git diff HEAD

# 编辑冲突文件
vi dev.txt

# 提交解决后的文件
git commit -am "解决冲突"
```
---
## 查看日志和分支图

```bash
# 查看提交历史图并设置别名
alias graph="git log --graph --online --decorate --all"
git graph
```
---
## Git进阶命令

### Git Stash 临时存储
- **把当前工作区和暂存区的改动临时保存起来**，然后让工作区恢复到**干净**的状态（和最近一次提交一致）-> 临时本地保存
```bash
# 保存当前修改(包括工作区和暂存区的修改),然后回到未提交状态(工作区和暂存区全部清空)
git stash save "备注" 

# 查看所有stash记录
git stash list

# 应用最近一次 stash，并且删除 stash 记录
git stash pop

# 应用最近一次 stash，但不删除 stash 记录
git stash apply
```
- 案例
```bash
# 保存当前改动 当前分支dev
git stash -m "测试" 
# 切换到别的分支操作
git checkout hotfix
# 切换回来
git checkout dev
# 恢复
git stash pop
```
### Git Rebase 分支基变

- 基变与合并的区别就是 将提交记录变成直线, 让历史更清晰简洁
![17443486993261744348698844.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17443486993261744348698844.png)
- [参考文献](https://blog.csdn.net/weixin_42310154/article/details/119004977)
### Git Reset 恢复回滚
- git reset比较常用的用法有两种
	- 一种是移动到指定的提交  git reset --soft/mixed/hard <commit_hash>
	- 一种是回退到上N个提交  git reset --soft HEAD~N
```bash
# 如果是软移动和软回退,不改变当前页面的工作区和暂存区,也就是保留当前的修改
git reset --soft abc123
git reset --soft HEAD^
# 如果是混合移动和混合回退,工作区修改保留,暂存区丢弃
git reset --mixed abc13
git reset --mixed HEAD^
# 如果是硬移动和硬回退,清空当前修改的暂存区和工作区
git reset --hard abc13
git reset --hard HEAD^
```
### Git Feach 拉取不合并
- `git fetch` 只是把远程仓库的更新下载到了本地 `.git` 目录中，不会直接影响本地的工作区和当前分支, 需要你**手动merger合并或者rebase基变**
- git pull则是**会获取所有远程索引并合并到本地分支**
### Git Reflog 本机历史记录
- 记录了**本地仓库**中分支的变化记录 和 HEAD的移动历史, `reflog` 是**本地记录**，**远程仓库没有**。
- 包括了- 提交（`git commit`）合并（`git merge`）变基（`git rebase`）撤销（`git reset`）拉取（`git pull`）checkout（切换分支）甚至删除分支也会有记录
```bash
# 通过reflog拿到上一次提交(不只是提交)的哈希值id 
git reflog

# 然后reset恢复到这次提交记录  
git reset --hard 哈希值

# 换一种方式    HEAD@{2}: 根据时间顺序排的第 n 次 HEAD 改变, 来源于历史记录
git reset --hard HEAD@{2}
```
### Git Cherry Pick 优选
- 将当前分支的提交复制一份到其他分支
```bash
# 比如当前在dev分支,有一个提交adc123,想把提交优选到main分支
git branch main 
git cherry-pick abc123
```
## Git 标签管理
```bash
# 创建标签
git tag v1.0.1

# 推送标签到远程
git push origin v1.0.1

# 删除本地标签
git tag -d v1.0.1

# 删除远程标签
git push origin :refs/tags/v1.0.1
```
## Git问答
- 问: 使用 `git merge` 的时候，Git 报错提示有一些本地的未跟踪（untracked）文件，而这些文件在合并进来的分支中也存在，导致无法完成合并, 如何解决?	
- 答: 使用git clean对不重要的本地untracked文件进行删除, 或者先把文件复制到其他地方备份, 或者通过git stash 进行保存, 或者通过git add 把文件交给版本管理

- 问: 如何忽略本地的某些文件? .gitignore 不生效?
- 答: 只有未被提交的文件才能真正被git忽略, 否则只要提交过一次就有了这个文件的追踪记录, 如果想真正让 Git 忽略已经被跟踪的 log/ 文件夹，需要先把它们从 Git 索引中移除，然后再让 .gitignore 生效
```java
git rm -r --cached logs // --cached:只从 Git 索引记录中删除,不动本地文件(本地文件还在,远程的索引和数据删除)
git commit -m // "把logs文件从版本库中移除"
git push origin main // 执行版本库中的logs文件的删除
```
