+++
date = '2025-12-30T23:23:13+08:00'
draft = false
title = 'Git'
+++

简单记录一下 **git** 的使用。

```bash
// 初始化一个仓库
git init
```

```bash
// 添加文件
git add .
```

```bash
// 可以查看当前版本库的状态
git status
```

```bash
// 可以放弃当前工作区的修改代码
git checkout -- <file> 
```

```bash
// 可以放弃 add 后，暂存区的代码
git reset HEAD <file>
git checkout -- <file>
```

只有在 github 仓库配置了ssh公钥之后才可以推送代码？
