---
title: 'Java多版本管理工具'
date: 2024-05-03
description: ' Java多版本管理工具'
# topic: leetcode
author: hzlei
# banner: https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/spring-boot-starter-security/index.webp
# cover: https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/spring-boot-starter-security/index.webp
nav_tabs: true
article:
  type:  # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: Java多版本管理工具。 # 必选
  # caption: 重点整理，包括一些框架特定的内容、特性，以及与其他框架的对比等。 # 标题下方的小字，可选
  color: hsl(93deg 48% 47%) # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...
---


## 安装

- Linux / OS X


```sh
git clone https://github.com/jenv/jenv.git ~/.jenv
```


- Mac OS X via Homebrew


```sh
brew install jenv
```


## 配置环境变量


- Bash

```sh
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(jenv init -)"' >> ~/.bash_profile
```


- Zsh

```sh
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc
```

## 添加 JDK

- 查看系统安装的 JDK

```sh
hzlei@mbp:~ $ ll /Library/Java/JavaVirtualMachines
total 0
drwxr-xr-x  3 root  wheel    96B  4月 10 10:48 amazon-corretto-17.jdk
drwxr-xr-x  3 root  wheel    96B  7月 10 03:44 amazon-corretto-21.jdk
drwxr-xr-x  3 root  wheel    96B  4月 12 22:13 amazon-corretto-8.jdk

```

- 添加

```sh
jenv add /Library/Java/JavaVirtualMachines/amazon-corretto-21.jdk/Contents/Home
```


## 命令详解

| 命令                 | 解释  |
|--------------------|-----|
| jenv versions      | 列出所有可用JDK版本 |
| jenv version       | 显示当前激活版本 |
| jenv version-name  | 仅打印版本号 |
| jenv global \<ver\>  | 设置系统全局默认的Java版本（写入 `~/.jenv/version` 文件） |
| jenv local \<ver\>   | 在当前目录设置Java版本（生成 `.java-version` 文件） |
| jenv shell \<ver\>   | 为当前Shell会话设置临时Java版本 |
| jenv add \<path>	   | 将已安装的JDK路径添加到jenv管理系统中 |
| jenv remove \<ver>	 | 从jenv中移除指定版本（不会卸载JDK文件） |
| jenv rehash        | 刷新命令垫片 |
| jenv doctor        | 环境健康检查 |
| jenv which \<cmd>   | 查找命令路径 |
| jenv whence \<cmd>   | 查找提供命令的版本 |




