---
title: Markdown 语法
date: 2017-12-05 21:03:34
tags: Markdown
categories: Tools
---

{% blockquote Wikipedia https://en.wikipedia.org/wiki/Markdown Markdown %}
Markdown is a lightweight markup language with plain text formatting syntax. It is designed so that it can be converted to HTML and many other formats using a tool by the same name.
{% endblockquote %}

最近在学习如何使用 [Hexo](https://hexo.io/) + [Github Pages](https://pages.github.com/) 搭建个人博客（其实是学习笔记）。Hexo 是使用 Markdown 解析文章，而自己还没怎么接触过 Markdown 的语法，于是，就有了这第一篇 note。

Windows 上可选择 [MarkdownPad](http://markdownpad.com/) 编辑器，不过我使用的是 [Visual Studio Code](https://code.visualstudio.com/)，配合 [Markdownlint](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint) 插件，感觉还不错！

## 标题

    # 一级标题
    ## 二级标题
    ### 三级标题
    #### 四级标题
    ##### 五级标题
    ###### 六级标题
    一级标题和二级标题还有一种写法，不推荐使用：
    Alt-H1
    ======（长度随意，下同）
    Alt-H2
    ------

一些规则：比如标题不能跨级，标题上下要有空行等等，具体可参考 [Markdownlint Rules](https://github.com/DavidAnson/markdownlint/blob/master/doc/Rules.md)。

<!-- more -->

## 强调

    Bold: **content** or __content__ (两个*或者两个_)
    Italic: *content* or _content_
    Strikethrough: ~~content~~
    Bold and italic: **content _content_ content**
    另外，以制表符或至少四个空格缩进的行会让 Markdown 保留所有的空白字符。本文中所有展示语法的代码区块就不会被渲染。

示例：**加粗**、_斜体_、~~删除线~~、**加粗 _斜体_ 加粗**、**bold _italic_ bold**

加粗和斜体都有两种表达方式，推荐采用 `**content**` 进行加粗，`_content_` 表示斜体。  
另外，中文情况下的加粗+斜体，斜体内容要加空格分开；英文单词间本来就有空格，所以不存在这个问题。

## 水平分区线

要生成水平分区线，可以在单独一行里输入3个或以上的短横线、星号或者下划线实现。短横线和星号之间可以输入任意空格。以下每一行都产生一条水平分区线。

    * * *
    ***
    *****
    - - -
    ---------------------------------------

## 引用代码

在段落中引用代码，比如 `git status`：

    `git status`

引用代码块，比如 JavaScript 代码：

````
```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```
````

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```

PS：如何不让 Hexo 解析代码块呢？解决办法是使用四个点将其包围，可参考 [HEXO 下的 Markdown 语法 (GFM) 写博客](https://zhuzhuyule.com/blog/HEXO/HEXO%E4%B8%8B%E7%9A%84Markdown%E8%AF%AD%E6%B3%95(GFM)%E5%86%99%E5%8D%9A%E5%AE%A2.html)，感谢 [@zhuzhuxia](https://github.com/zhuzhuyule) 这位老哥。

## 链接

第一种方式：This site was bulit using [GitHub Pages](https://pages.github.com/). 推荐采用这种方式。

    This site was bulit using [GitHub Pages](https://pages.github.com/).

另一种方式和参考文献的写法类似：[Hexo] is a fast, simple & powerful blog framework。  
一般将 `[name]: link` 键值对放在文档末尾（ name 不区分大小写），当多个地方用到同一个 link 时，可使用这种方式。

    [Hexo] is a fast, simple & powerful blog framework.
    ...
    content
    [Hexo]...
    content
    ...
    [Hexo]: https://hexo.io/

## 列表

### 无序列表

无序列表使用 `+` 、 `-` 、 `*` 作为列表标记，一般采用相同的符号，记得后面加个空格。

    * George Washington
    * John Adams
    * Thomas Jefferson

* George Washington
* John Adams
* Thomas Jefferson

### 有序列表

有序列表采用数字 `1.`、 `2.`、 `3.` 作为列表标记，同样要加空格。

    1. James Madison
    2. James Monroe
    3. John Quincy Adams

1. James Madison
2. James Monroe
3. John Quincy Adams

### 内嵌列表

当列表包含子列表时，[Markdownlint Rules] 的 MD004 要求子列表缩进两个空格，且 sublist 样式下（设置 `"MD004": { "style": "sublist" }` ），只需保证同级子列表使用相同的符号。

    * Item 1
      + Item 2
        - Item 3
      + Item 4
    * Item 5
      + Item 6

* Item 1
  + Item 2
    - Item 3
  + Item 4
* Item 5
  + Item 6

## 图片

图片和[链接](#链接)的语法差不多，只是前面多了一个 `!`，这里只介绍第一种形式。

    ![Hexo icon](https://hexo.io/icon/favicon-96x96.png)

hexo的icon:
![Hexo icon](https://hexo.io/icon/favicon-96x96.png)

## 引用

引用时在段落前加上 `>` 作为标记。当引用多个段落时，最好在空白行也加上 `>`，使之成为一个整体。

    > His words seemed to have struck some deep chord in his own nature. Had he spoken of himself, of himself as he was or wished to be? Stephen watched his face for some moments in silence. A cold sadness was there. He had spoken of himself, of his own loneliness which he feared.
    >
    > —Of whom are you speaking? Stephen asked at length.
    >
    > Cranly did not answer.

> His words seemed to have struck some deep chord in his own nature. Had he spoken of himself, of himself as he was or wished to be? Stephen watched his face for some moments in silence. A cold sadness was there. He had spoken of himself, of his own loneliness which he feared.
>
> —Of whom are you speaking? Stephen asked at length.
>
> Cranly did not answer.

## 段落

两段之间用空行隔开，这是所谓的 hard break。还有一种 soft break，即在前一段之后加两个空格，常用来格式化列表中的多个段落。

    1. Crack three eggs over a bowl.

        Now, you're going to want to crack the eggs in such a way that you don't make a mess.(两个空格)
        If you _do_ make a mess, use a towel to clean it up!

    2. Pour a gallon of milk into the bowl.

        Basically, take the same guidance as above: don't be messy, but if you are, clean it up!

1. Crack three eggs over a bowl.  
    Now, you're going to want to crack the eggs in such a way that you don't make a mess.  
    If you _do_ make a mess, use a towel to clean it up!
2. Pour a gallon of milk into the bowl.  
    Basically, take the same guidance as above: don't be messy, but if you are, clean it up!

## 表格

使用 `|` and `-` 制作表格。

    | First Header  | Second Header |  
    | ------------- | ------------- |  
    | Content Cell  | Content Cell  |  
    | Content Cell  | Content Cell  |  

| First Header  | Second Header |
| ------------- | ------------- |
| Content Cell  | Content Cell  |
| Content Cell  | Content Cell  |

单元格的长度不必和表头的长度相同。单元格的内容可加一些样式，比如加粗、斜体等。

    | Command | Description |  
    | --- | --- |  
    | `git status` | List all _new or modified_ files |  
    | `git diff` | Show file differences that **haven't been** staged |  

| Command | Description |
| --- | --- |
| `git status` | List all _new or modified_ files |
| `git diff` | Show file differences that **haven't been** staged |

默认表头居中（hiker 主题表头左对齐），内容左对齐。可使用 `:` 修改每一列的对齐方式，如下所示。另外，如果单元格中需要加入字符 `|`，可通过 `\|`转义。

    | Left-aligned | Center-aligned | Right-aligned |
    | :---         |     :---:      |          ---: |
    | git status   | git status     | git status    |
    | \|           | \|             | \|            |

| Left-aligned | Center-aligned | Right-aligned |
| :---         |     :---:      |          ---: |
| git status   | git status     | git status    |
| \|           | \|             | \|            |

PS：Hexo 默认使用 [hexo-renderer-marked](https://github.com/hexojs/hexo-renderer-marked) 解析，转义 `\|` 时会出错，解决办法是改用 [hexo-renderer-markdown-it](https://github.com/hexojs/hexo-renderer-markdown-it)。
但写本文时 npm 中 hexo-renderer-markdown-it 的版本为3.4.1（可通过 `npm show <package-name> version` 查看版本号）， 必须通过 git 安装：

```
 npm install git+ssh://git@github.com:hexojs/hexo-renderer-markdown-it.git
```

另外，解决 hexo-renderer-markdown-it 不能识别 `<!-- more -->` 的方法参见 [Issue #14](https://github.com/hexojs/hexo-renderer-markdown-it/issues/14)。

## 推荐阅读

* [Basic writing and formatting syntax](https://help.github.com/articles/basic-writing-and-formatting-syntax/) in [writing-on-github](https://help.github.com/categories/writing-on-github/)

* [Markdown: Syntax](https://daringfireball.net/projects/markdown/syntax)

* [Markdown 书写风格指南](http://einverne.github.io/markdown-style-guide/zh.html)

* [Markdown 语法说明 (简体中文版)](http://wowubuntu.com/markdown/)

* [Markdownlint Rules](https://github.com/DavidAnson/markdownlint/blob/master/doc/Rules.md)

* [Markdown Tutorial](https://www.markdowntutorial.com/)

[Hexo]: https://hexo.io/
[Markdownlint Rules]: https://github.com/DavidAnson/markdownlint/blob/master/doc/Rules.md