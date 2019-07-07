---
title: Hexo + GitHub Pages 搭建个人博客
date: 2017-12-15 14:35:42
tags: Hexo
categories: Tools
---

{% blockquote Documentation https://hexo.io/docs/index.html hexo.io %}
Hexo is a fast, simple and powerful blog framework. You write posts in Markdown (or other languages) and Hexo generates static files with a beautiful theme in seconds.
{% endblockquote %}

这一篇 note 讲解如何使用 [Hexo](https://hexo.io/) + [Github Pages](https://pages.github.com/) 搭建个人博客，并用 GitHub 进行版本控制。其中源文件位于 hexo 分支，静态文件位于 master 分支。

## 准备

* 安装最新版的 [Git](https://git-scm.com/)。在命令行输入 `git version` 检查 git 是否安装成功。

* 安装 LTS 版的 [Node.js](https://nodejs.org/en/)。同样在命令行输入 `node -v` 和 `npm -v`以检查 node.js 是否安装成功。

* 注册 GitHub 账号，新建一个 repository，一般命名为 `username.github.io`，这样 GitHub 会自动开启 GitHub Pages 功能。勾选 `Initialize this repository with a README` 的话即可访问个人主页，否则需要添加内容才能访问，建议暂时不要勾选。

* GitHub 添加 SSH（推荐），可参考 [Connecting to GitHub with SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)。

<!-- more -->

## Hexo

### 安装 Hexo

```
npm install hexo-cli -g
```

使用 `hexo version` 检查是否安装成功。

### 建站

选择一个目录，比如 D:\github，键入以下命令：

```
hexo init username.github.io
cd username.github.io
npm install
```

这样就创建了一个名为 `username.github.io` 的 Hexo 工程（文件夹）。注意，`hexo init <folder>` 命令要求 folder 为空文件夹，否则会报错。

### 启动服务器

```
hexo server  (简写为 hexo s)
```

默认情况下，访问网址为 `http://localhost:4000/`。打开网址，可以看到一篇 landscape 主题的 Hello World 博客。一般修改 Markdown 文件，不需要重启服务器，直接刷新浏览器即可，除非你修改配置文件。

### 修改配置

#### 站点配置

工程根目录下的 **_config.yml** 称为站点配置文件，可以配置一些个人信息等，具体可参考[Configuration](https://hexo.io/docs/configuration.html)。

#### 主题配置

每个主题的目录下也会有一个 **_config.yml** 文件，称为主题配置文件。Hexo 有丰富多彩的主题，这里以 [hexo-theme-hiker](https://github.com/iTimeTraveler/hexo-theme-hiker) 为例，说明如何更换主题。

安装主题：

```
cd username.github.io
git clone git@github.com:iTimeTraveler/hexo-theme-hiker.git themes/hiker
```

PS：这样安装主题并不能 push 到 GitHub 中去，可使用 `fork + subtree` 的方法解决，具体参考 [Hexo 主题同步](http://w4lle.com/2016/06/06/Hexo-themes/)。感谢 [@Tyrion Yu](https://github.com/tyrionyu) 的帮助。

修改站点配置文件，将 theme 修改为 hiker：

    # Extensions
    ## Plugins: https://hexo.io/plugins/
    ## Themes: https://hexo.io/themes/
    theme: hiker

重启服务器，即可查看效果。在某些情况（尤其是更换主题后），如果发现对站点的更改无论如何也不生效，可能需要 clean 一下。

```
hexo clean
hexo server
```

PS：如果不需要 landscape 主题，直接删除 themes 下的对应文件夹即可。

#### 部署配置

安装 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)。

```
npm install hexo-deployer-git --save
```

然后修改站点配置文件：

    # Deployment
    ## Docs: https://hexo.io/docs/deployment.html
    deploy:
      type: git
      repo: git@github.com:username/username.github.io.git  # 这种配置需使用 SSH
      branch: master
      message: message  # 默认为 Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }}

其中 branch 为静态文件所在的分支，[必须为 master 分支](https://help.github.com/articles/user-organization-and-project-pages/)。message 表示自定义提交信息，一般不需要配置，删除该行即可。

### Git

#### Git 初始化

为工程创建 Git 仓库：

```
cd username.github.io
git init
```

#### 创建分支

此时 Git 仓库为空，不能直接运行 `git branch hexo` 来创建新的分支，可通过以下命令创建：

```
git checkout -b hexo
```

#### push 源文件

添加所有文件，提交到本地仓库：

```
git add .
git commit -m "first commit"
```

添加远程仓库，并push：

```
git remote add origin git@github.com:username.github.io.git
git push -u origin hexo
```

#### 部署静态文件

先生成静态文件，再部署：

```
hexo clean
hexo generate  （简写为 hexo g）
hexo delpoy  (简写为 hexo d)
```

此时整个部署过程就结束啦，你可以通过 `https://username.github.io/` 访问自己的 Github Pages。

## 推荐阅读

* [How to use Hexo and deploy to GitHub Pages](https://gist.github.com/btfak/18938572f5df000ebe06fbd1872e4e39)

* [Hexo Documentation](https://hexo.io/docs/)

* [知乎：使用hexo，如果换了电脑怎么更新博客？](https://www.zhihu.com/question/21193762)

* [An attractive theme for Hexo. called "Hiker", short for "HikerNews"](https://github.com/iTimeTraveler/hexo-theme-hiker)

* [Hexo 主题同步](http://w4lle.com/2016/06/06/Hexo-themes/)

* [使用 git subtree 集成项目到子目录](https://aoxuis.me/bo-ke/2013-08-06-git-subtree)

* [Git subtree: the alternative to Git submodule](https://www.atlassian.com/blog/git/alternatives-to-git-submodule-git-subtree)