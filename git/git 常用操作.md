# Git 常用操作

## 查看提交历史
git log

## 将某个分支合并到另一个分支
git checkout [targetBranch]
git merge [srcBranch]

## 合并某个特定的分支
git cherry-pick {commitId}

步骤：
1. git log 查看提交的 commit id
2. git checkout {branch}
3. git cherry-pick {commitId}

## 合并多个提交
git rebase -i [提交id]  合并提交号前的提交

## 重写 commit message
git commit --amend

## 撤销提交
git reset [--hard|--soft|--mixed] [提交id]
- -- hard：彻底回退到指定版本，暂存区和index都会清空
- --soft：只回退 commit，不回退index
- --mixed：保留源代码，回退 commit 和 index

## tag 相关的命令
git tag：查看所有的tag
git tag xxxx：创建 tag xxxx

## remote 命令
git remote add origin url：绑定远程仓库
git remote remove origin：取消关联远程仓库

## 生成 ssh 文件
ssh-keygen -t rsa -C "503871010@qq.com" -f id_rsa.coding
