## 1. 本地仓库如何绑定远端仓库
1. git push时指定
```bash
# 1. 添加远程仓库地址
git remote add origin https://github.com/你的用户名/仓库名.git

# 2. 检查远程仓库是否添加成功
git remote -v

# 3. 将本地分支重命名为 main（与 GitHub 默认一致）
git branch -M main

# 4. 推送到 GitHub（-u 表示设置上游分支，下次可直接 git push）
git push -u origin main

# 如果remote更换地址，可以删除后重新添加
git remote remove origin
```
2. 手动指定
```bash
# 为当前分支设置追踪的远程分支
git branch --set-upstream-to=origin/main

# 简写
git branch -u origin/main

# 取消追踪
git branch --unset-upstream

# 本地分支
git branch

# 远程分支
git branch -r

# 所有分支（含追踪关系）
git branch -vv

```
3. git push 时指定独立分支
```bash
# 本地分支叫 feature-login，可以推送到远程的 main
git push origin feature-login:main

# 本地叫 dev，远程叫 develop
git push origin dev:develop

```
## 2. 针对not staged、deleted和untrack状态的文件处理
1. not staged
```bash
# 查看具体哪些文件被修改
git status

# 丢弃所有修改（包括删除的文件）
git checkout -- .
# 或
git restore .

# 仅丢弃某个文件的修改
git checkout -- 文件名
# 或
git restore 文件名
```

2. deleted

```bash
 # 恢复误删的文件（从 Git 历史还原）
git checkout -- 被删除的文件名

# 或确认要删除，添加到暂存区
git add -A    # 将所有删除标记为已暂存
git commit -m "删除文件"
```

3. untracked
```bash
# 查看哪些未跟踪文件将被删除（预览模式，安全）
git clean -n

# 删除未跟踪的文件（不包括文件夹）
git clean -f

# 删除未跟踪的文件 + 文件夹
git clean -fd

# 连 .gitignore 忽略的文件也一起删（最彻底）
git clean -fdx
```