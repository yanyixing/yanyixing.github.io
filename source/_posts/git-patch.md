---
title: git_patch
date: 2018-03-08 14:01:19
tags: git
---

# git format-patch

适用于git的patch，包含diff信息，包含提交人，提交时间等 如果<font color=red>git format-patch</font>生成的补丁不能打到当前分支，git am会给出提示，并协助你完成打补丁工作

## 对比分支生成patch

例：从master checkout 一个新分支修改然后与master对比生成patch。

```bash
$ git format-patch -M master   # -M选项表示这个patch要和那个分支比对
$ git am 001-xxx.patch         # 不必重新commit
```

## 将commit打包成patch

```bash
$ git format-patch 1bbe3c8c19 # 从1bbe3c8c19往后的全部commit,不包括1bbe3c8c19

git format-patch -n 1bbe3c8c19 # 1bbe3c8c19开始n个commit，包含1bbe3c8c19

$ git format-patch HEAD^      # 最近的1次commit的patch
$ git format-patch HEAD^^     # 最近的2次commit的patch
$ git format-patch HEAD^^^    # 最近的3次commit的patch

$ git format-patch -1         # 同HEAD^  
$ git format-patch -2         # 同HEAD^^
$ git format-patch -3         # 同HEAD^^^

$ git format-patch -1 -4      # -1到-4之间的commit，包括-1和-4

$ git format-patch <ref1>..<ref2>  ## 在两个commit之间，包括ref1和ref2 
```

## patch合并到一个文件

默认情况下，每一个commit都会产生一个patch文件，下面的操作可以把全部commit合并到一个文件中

```bash
$ git format-path 1bbe3c8c19 --stdout > xxx.patch
```

## 应用patch

```bash
# 检查patch文件
$ git apply --stat xxx.patch

#查看是否能应用成功
$ git apply --check xxx.patch

# 应用patch
$ git am -s < xxx.patch
```