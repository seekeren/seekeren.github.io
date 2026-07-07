---
date: '2026-07-06T19:14:33+08:00'
draft: false
title: '消除git diff中的敏感信息'
tags: ["git", "github"]
---

# 起因
起因是这样的，我在GitHub新开了一个项目，是智能体相关的，代码中写明了一个敏感值，我意识到之后赶紧删除了，然后新提交了。
但是，git的diff可以查看代码的改动处，所以还是能看到这个敏感值。敏感信息还是要不可见的。
大致的解决办法如下，通过一个git的过滤仓库的工具，把敏感信息替换成一个默认值，先把本地的提交历史全部改一遍，然后同步到远程仓库。
git是真牛啊，连我这样的需求都能满足，强。

# 解决过程
安装filter-repo

```bash
pip install git-filter-repo
```

在项目根目录下创建一个replace.txt文件，用于正则表达式替换敏感值。在TXT文件中写入如下内容，意思就是将api_key的值替换为REDACTED，并支持等号两边的空格。
```bash
regex:api_key\s*=\s*.+==>api_key=REDACTED
```

```bash
# 全仓库所有文件、所有提交替换敏感字符
git filter-repo --replace-text replace.txt
```

如果仅需要限定某个文件做替换:
```bash
git filter-repo --path config/application.yml --replace-text replace.txt
```
一般执行完这步之后，会清除与远程仓库的关联，所以需要重新关联一下远程仓库的地址。我是从idea的manage remote添加的。

校验：(可选)
```bash
# 全局搜索所有历史是否还有敏感明文
git log --all -S "abc123456"
# 无输出=清理成功；有输出说明规则没匹配到，重新调整replace.txt

# 随机查看旧提交diff验证
git log -p --all | grep "abc123456"
```

推送所有分支：
```bash
# 推送所有分支
git push origin --force --all
# 推送所有标签
git push origin --force --tags
```



