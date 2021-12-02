---
title: 更换电脑后，如何继续使用hexo更新博客？
date: 2017-11-15 16:59:54
categories: [网站建设]
tags: [Hexo]
cover: cover.jpg
---
最近在使用hexo更新博客的过程中，发现一个比较麻烦的事情，就是修改配置文件后，不能便捷的把配置同步到其它电脑上，直接拷贝配置文件到其它电脑显然不是一个明智的选择，总有出错的时候，并且没法追溯历史修改，于是，用git的分支来管理hexo的源码和静态文件便是不二的选择，简单总结一下。
<!-- more -->


### 初始搭建流程
- 创建仓库之后创建master（放静态文件）和hexo（放源码）两个分支；
- 使用`git clone -b hexo 仓库地址` 命令拷贝 `hexo` 分支到本地；
- 进入本地仓库文件夹，保持在hexo分支，依次执行`npm install hexo`、`hexo init`、`npm install` 和 `npm install hexo-deployer-git`；
- 根据需要修改hexo配置文件，注意deplay参数的分支应该为master；
- 依次执行`git add .` 、`git commit -m "..."` 、`git push origin hexo` 把源码提交到远程hexo分支；
- 接着执行 `hexo d -g` 部署网站到GitHub Pages。

这样就是一个完整的流程了，你会发现，hexo自动生成的文件里面已经有一个`.gitignore`文件，说明其本意就是希望我们把源码存放到github，这个文件里面已经帮我们设置好了哪些文件不应该被提交，如下：
````
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
````
> 注： `themes` 文件夹因为一次修改好之后几乎不会有什么变动了，可以跟随源码一起存放到gitgub，这样换电脑的时候就不用再配置主题，如果需要换新的主题，提交源码的时候把新的主题包加进去即可。

### 日常更新流程
本地更新博客（文章及样式修改）后，按照下面两步走来进行代码更新。
- 依次执行 `git add .`、`git commit -m "..."`、`git push origin hexo`更新源码到远程hexo分支；
- 执行 `hexo d -g` 部署网站到GitHub Pages。

> 注：建议严格按照以上两个步骤来执行，如果先执行第2步，在极端情况下（比如突然停电、死机），会导致源码部分丢失。

### 更换电脑后的流程
换电脑后，按照以下步骤操作：
- 使用`git clone -b hexo 仓库地址 ` 命令拷贝 `hexo` 分支到本地；
- 进入本地仓库文件夹，保持在hexo分支，依次执行`npm install hexo`、`npm install`；

>注：这里不需要 `hexo init` 命令了，另外，平时安装插件的时候记得加上 `--save` 参数，这样 `package.json`文件里就会写入插件的安装信息，当我们换电脑的时候，只需要 `npm install` 命令即可安装所有插件。

最后说一下，因为github pages要求代码库不能是私有的，如果你觉得用这种方式保存源码不够安全的话，可以重新创建一个私有库专门保存源码，使用流程上也不会有太大的变化。