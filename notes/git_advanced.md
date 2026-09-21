# Git 进阶

## 撤销工作区改动
    git checkout -- <file>        # 丢弃单个文件改动
    git restore <file>            # 新版推荐写法

## 改写最近一次提交
    git commit --amend -m "新信息"

## 把 A 仓的某个目录搬进 B 仓（保留历史）
    git filter-repo --path <dir>  # 需 git-filter-repo

## 子模块
    git submodule add <url> <path>
    git submodule update --init --recursive

## 临时藏起来
    git stash && git stash pop
