---
title: 自动部署Hexo站点至Github
tags:
  - Hexo
  - Github
  - Travis CI
date: 2017-02-21 15:20:46
---

前文 {% post_link Hexo+Github快速搭建博客 %} 中提到了如何使用Hexo搭建自己的博客，但每次的部署还需要额外进行一些操作，既然作为程序员，就使用自动集成系统来完成这最后的重复性操作吧。

<!-- more -->

## 了解Travis CI

> Travis CI是在软件开发领域中的一个在线的，分布式的持续集成服务，用来构建及测试在GitHub托管的代码。

其完整的构建周期如下：

1. `apt addons` (可选)
2. `cache components` (可选)
3. `before_install`
4. `install`
5. `before_script`
6. `script`
7. `before_cache` (可选)
8. `after_success` 或 `after_failure`
9. `before_deploy` (可选)
10. `deploy` (可选)
11. `after_deploy` (可选)
12. `after_script`

## 方案选择

目前Travis CI所支持的部署方案大致分为以下几种：

- [Github Releases](https://docs.travis-ci.com/user/deployment/releases/)
- Personal Access Token
- Encrypting File + SSH Key

其中，官方支持的Github Releases方式最为简单，但仅支持Tag而不支持分支的推送方式，不足以满足Hexo的部署需求。
使用Token的部署，可以选择在自动集成时候修改`_config.yml`中Github地址并使用Hexo命令部署，或在`script`阶段通过Git推送`public`目录的方式实现。
为了避免与项目结构、配置耦合，最终选择了SSH Key的方式。

## 配置

### 注册Travis CI

首先，可以使用Github账号直接注册一个[Travis CI](https://travis-ci.org)账号，并打开对应的仓库的自动集成功能。

### 准备SSH Key

创建SSH Key

```bash
$ ssh-keygen -t rsa -C "your_email@example.com"
```

视情况选择保存路径，此处选择了`/tmp/ssh_key`，`passphrase`留空即可。

```bash
Enter a file in which to save the key (/Users/you/.ssh/id_rsa): /tmp/ssh_key
Enter passphrase (empty for no passphrase): [Press enter]
Enter same passphrase again: [Press enter]
Your identification has been saved in /tmp/ssh_key.
Your public key has been saved in /tmp/ssh_key.pub.
```

复制公钥内容至Github账户，[Github说明文档](https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/)。

### 加密私钥

使用Travis命令行工具加密私钥，[Encrypting Files说明文档](https://docs.travis-ci.com/user/encrypting-files/)。
若未在项目中执行`encrypt-file`命令，会提示"Can't figure out GitHub repo name"，此时可通过`-r <owner>/<repo>`参数手动指定仓库名称，该命令将解密用的KV对上传至Travis CI的后台，可在对应项目的Settings中找到相关的环境变量。

```bash
$ gem install travis
$ travis login --auto
$ travis encrypt-file /tmp/ssh_key
```

找到输出中包含KV对的内容并复制
```
openssl aes-256-cbc -K $encrypted_<id>_key -iv $encrypted_<id>_iv -in ssh_key.enc -out /tmp/ssh_key -d
```

最后，将加密后的私钥拷贝至项目中，自行保管或删除私钥及公钥的原文件。

```bash
$ mkdir .travis
$ mv ssh_key.enc .travis/
# 此处删除了私钥及公钥
$ rm /tmp/ssh_key /tmp/ssh_key.pub
```

### 配置SSH Config

于`.travis/`目录建立`ssh_config`文件。

```
Host github.com
  User git
  IdentityFile ~/.ssh/id_rsa
  StrictHostKeyChecking no
  PasswordAuthentication no
  BatchMode yes
```

### 配置.travis.yml

修改之前复制的包含KV对的命令，并建立`.travis.yml`文件。
注意配置中的`branches`项需要根据Hexo仓库配置做对应调整。

```yaml
language: node_js
node_js: stable

branches:
  only:
    - develop

before_install:
  - openssl aes-256-cbc -K $encrypted_<id>_key -iv $encrypted_<id>_iv -in .travis/ssh_key.enc -out ~/.ssh/id_rsa -d
  - chmod 600 ~/.ssh/id_rsa
  - cp .travis/ssh_config ~/.ssh/config
  - git config --global user.name "your_name"
  - git config --global user.email "your_email@example.com"

install:
  - npm install -g hexo-cli
  - npm install --quiet

script:
  - hexo generate
  - hexo deploy
```

## 提交代码

将`.travis`目录及`.travis.yml`提交并推送至代码仓库，查看Travis CI日志检查是否成功部署，或定位相关问题。
如最终日志如下提示，则表示已经成功部署：

```
The command "hexo deploy" exited with 0.

Done. Your build exited with 0.
```

此后的每次提交，Travis CI便能自动集成并部署至Github了。


## 参考资料

- [SSH Key 相关介绍](https://help.github.com/articles/connecting-to-github-with-ssh/)
- [Travis CI 帮助文档](https://docs.travis-ci.com/user/getting-started/)
