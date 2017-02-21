---
title: Hexo+Github快速搭建博客
date: 2016-12-30 00:00:00
tags:
  - Hexo
  - Github
---

## 简介

想要搭建一个博客，但又不习惯于各种富文本编辑器；
在网速与备案之间犹豫不决，或觉得VPS依然太贵；
那么就使用Hexo+Github快速搭建一个静态博客吧。

> 类似的解决方案还有：Jekyll, Octopress, Medium, 简书, FarBox 等等。

<!-- more -->

## 安装

### 前置安装

这些工具的安装方式比较常见并且与系统相关，就不做详细说明了。

- Node.js
- Git

### 操作

#### 安装Hexo

``` bash
$ npm install -g hexo-cli
```

#### 建站

``` bash
$ hexo init my-blog
$ cd my-blog
$ npm install

# 建议安装Git部署插件
$ npm install hexo-deployer-git --save
```

#### 写作

``` bash
$ hexo new post <title>
```

然后就能在`./source/_posts/<title>.md`找到并使用Markdown语法编辑文章了。
其中由文件最上方以`---`分隔的区域为文章的属性，详细内容可参考[Hexo Front-matter](https://hexo.io/zh-cn/docs/front-matter.html)。

#### 预览

``` bash
$ hexo server
```

然后打开浏览器访问`http://localhost:4000/`即可访问。


#### 更换主题

以下以Next为例进行更换主题。

``` bash
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
# 此时可删除原本的默认主题
```

然后修改`_config.yml`，重启服务以预览新主题。
主题的配置文件位于`themes/next/_config.yml`,详细内容可参考[Next 主题设定](http://theme-next.iissnan.com/getting-started.html#theme-settings)

```
theme: next
```

#### 配置并部署

正式发布站点前，还需要进行一些设置，详细内容可参考[Hexo 配置](https://hexo.io/zh-cn/docs/configuration.html)。

```
deploy:
  type: git
  repo: <Your repository url>
  branch: master
```

上述设置中的branch为部署分支，与写作分支是相互独立的，观察默认的`.gitignore`文件就能够明白其工作原理。
建议使用`develop`分支进行写作及版本控制，`master`分支进行部署。如果使用的不是同名仓库，可以使用`master`分支写作，`gh-pages`分支发布。

```bash
$ git checkout -b develop
```

确认文章完成并且设置正确后，使用以下命令发布博客：
``` bash
$hexo generate --deploy
```


## 参考资料

- [Hexo 主页](https://hexo.io/zh-cn/)
- [Next 主页](http://theme-next.iissnan.com/)
