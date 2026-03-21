```markdown
# Git 常见问题解答

---

### 如何修改已经提交的 commit message？

## 答案

根据提交是否已推送到远端，修改方式有所不同。

**① 修改最近一次提交（未推送）**

```bash
git commit --amend -m "新的提交说明"
```

**② 修改更早的提交（交互式 rebase）**

```bash
# 回溯最近 N 次提交，N 替换为数字
git rebase -i HEAD~N

# 在编辑器中，将要修改的那行 pick 改为 reword（或 r）
# 保存后 Git 会再次打开编辑器让你填写新的 message
```

**③ 已推送到远端：强制推送（慎用）**

```bash
git push --force-with-lease origin branch-name
```

> ⚠️ 修改已推送的历史会改变 commit hash，会影响协作者，务必与团队沟通后再操作。建议使用 `--force-with-lease` 而非 `--force`，前者更安全。

---

### Git 如何查看文件？

## 答案

Git 提供多种维度查看文件内容与差异。

| 场景 | 命令 | 说明 |
|------|------|------|
| 查看某次提交中的文件内容 | `git show <hash>:path/to/file` | 输出该文件在指定 commit 时的快照 |
| 查看工作区与暂存区的差异 | `git diff path/to/file` | 尚未 add 的改动 |
| 查看暂存区与上次提交的差异 | `git diff --staged path/to/file` | 已 add、未 commit 的改动 |
| 查看两次提交之间的差异 | `git diff <hash1> <hash2> -- file` | 对比任意两个版本 |
| 列出某次提交改动的文件清单 | `git show --stat <hash>` | 只看文件名和增删行数 |
| 追踪文件每行的最后修改者 | `git blame path/to/file` | 逐行显示作者与 commit |
| 查看文件的历史变更记录 | `git log --follow -- path/to/file` | `--follow` 可跟踪重命名 |

---

### Git 工作区和暂存区的区别？HEAD 是什么？

## 答案

Git 将文件管理划分为三个层次，一次完整的提交流程会依次经过这三层：

```
工作区 (Working Directory)
    │  你在磁盘上实际编辑的文件目录
    │  改动状态：Untracked / Modified
    │
    │  git add
    ▼
暂存区 (Staging Area / Index)
    │  .git/index 文件，是一块"预发布区"
    │  存放下一次提交的快照
    │  改动状态：Staged
    │
    │  git commit
    ▼
本地仓库 (Repository / .git)
    │  永久记录的提交历史 (commit objects)
    │
    └──▶  HEAD  ←── 当前所在的提交或分支
```

- **工作区**：你可以自由读写的普通文件目录，Git 只是监控它的变化，不会主动保存快照。
- **暂存区**：执行 `git add` 后，文件的当前状态被"登记"到这里。一次 commit 只打包暂存区的内容，这让你可以精细控制"提交哪些改动"。
- **HEAD**：一个指向"当前位置"的特殊指针。正常情况下它指向当前分支的最新 commit；当你 `git checkout <hash>` 时，HEAD 直接指向某个 commit，此时称为"游离 HEAD（detached HEAD）"状态。

```bash
# 查看 HEAD 当前指向
cat .git/HEAD
# 输出示例：ref: refs/heads/main

# 查看三个区域的文件状态
git status
```

---

### Git Rebase 是什么？如何使用？

## 答案

`git rebase` 的核心作用是：**将一条分支上的提交"搬运"到另一个基点之上**，从而得到线性、整洁的提交历史。

**与 merge 的对比**

```
── Merge（产生 merge commit，保留分叉历史）

  A───B───C  (main)
       \     \
        D─E───M  ← merge commit

── Rebase（线性历史，无 merge commit）

  A───B───C  (main)
               \
                D'──E'  ← 搬运后的提交（hash 已重写）
```

**基本用法**

```bash
# 在 feature 分支上，将其"嫁接"到 main 最新提交之上
git checkout feature
git rebase main

# 遇到冲突时：解决后继续
git add .
git rebase --continue

# 中途放弃 rebase
git rebase --abort
```

**交互式 rebase（整理提交历史）**

```bash
git rebase -i HEAD~3   # 整理最近 3 次提交

# 编辑器中可用的操作：
# pick   保留该提交
# reword 修改 commit message
# squash 合并到上一个提交
# fixup  合并到上一个提交（丢弃本条 message）
# drop   删除该提交
```

> ⚠️ Rebase 会重写 commit hash。**不要对已推送到远端且他人正在使用的分支执行 rebase**，否则会造成历史冲突。
>
> 💡 黄金法则：只对本地或私有分支 rebase，公共分支用 merge。

---

### git push 与 git fetch 的区别？

## 答案

两个命令方向相反：`fetch` 是"取回"，`push` 是"推送"；且对工作区的影响也截然不同。

```
本地仓库                         远端仓库 (origin)
─────────────────                ─────────────────
  工作区
    │
  暂存区
    │
  本地分支 (main)  ─── git push ───▶  origin/main
                   ◀── git fetch ──  origin/main
  远端跟踪分支
  (origin/main)

  git pull = git fetch + git merge（或 rebase）
```

| 维度 | git fetch | git push |
|------|-----------|----------|
| 方向 | 远端 → 本地 | 本地 → 远端 |
| 影响工作区？ | ❌ 不影响，只更新远端跟踪分支 | ❌ 不影响工作区，只写远端 |
| 自动合并？ | ❌ 不合并，需要手动 merge/rebase | —（推送到远端，不涉及合并） |
| 安全性 | ✅ 非常安全，只读操作 | ⚠️ 强制推送 (--force) 有风险 |
| 典型场景 | 先看看远端有哪些新内容，再决定如何整合 | 将本地提交同步到远端共享 |

**推荐的安全工作流**

```bash
# 拉取远端更新（更安全的方式，先 fetch 再手动 merge）
git fetch origin
git log HEAD..origin/main  # 查看新增的提交
git merge origin/main      # 确认没问题后再合并

# 推送到远端
git push origin main

# 推送并设置上游（首次推送新分支时）
git push -u origin feature-branch
```

> 💡 `git pull` = `git fetch` + `git merge`（默认）。若想更细粒度地控制合并时机，建议养成先 `fetch` 后手动合并的习惯。
```