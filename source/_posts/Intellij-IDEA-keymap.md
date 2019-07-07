---
title: IntelliJ IDEA Tips and Tricks
date: 2018-03-03 08:40:41
tags: IntelliJ IDEA
categories: Tools
---

{% blockquote IntelliJ IDEA Blog https://blog.jetbrains.com/idea/2015/10/intellij-idea-tips-and-tricks/ IntelliJ IDEA Tips and Tricks %}
Three weeks ago we announced IntelliJ IDEA Preview and now we are preparing for the release. Hadi Hariri, Developer Advocacy Lead at JetBrains, presented his talk at JavaZone entitled ‘IntelliJ IDEA Tips ans Tricks’ which also includes features from IntelliJ IDEA 15.
{% endblockquote %}

{% youtube eq3KiAH4IBI %}

<!-- more -->

## Tips and Tricks

IntelliJ IDEA 是一款优秀的 Java IDE，其便捷操作性，快捷键的功劳占了一大半。IntelliJ IDEA 本身的设计思维是提倡键盘优先于鼠标的，所以各种快捷键组合层出不穷，对于快捷键设置也有各种支持，对于其他 IDE 的快捷键组合也有预设模板进行支持。前段时间看了 Hadi Hariri 的演讲，打算把其中的一些 Tips 记录下来，加深记忆。

* Presentation assistance plugin

    演示助手插件，可以显示操作的名称和 Win/Mac 下对应的快捷键。例如打开设置 `Ctrl + Alt + S` 就会在屏幕下方显示 `Settings via Ctrl + Alt + S (Cmd + 逗号 for Mac)`

* Working without editor tabs

    Settings -> Editor -> General -> Editor Tabs，设置 Placement 为 None。可以用下文 Navigation 中的快捷键转到另一个文件。

* Autoscroll from/to source

    使用 Autoscroll from source，编辑哪个文件，Project window 便会定位到该文件所在的目录。具体设置很简单，单击 Project 窗口右上角的设置图标，选中 Autoscroll from source 即可。

* Create new file/folder

    `Alt + 1` 激活 Project 窗口， 或者`Alt + Home` (笔记本上的 Home 键一般要配合 Fn 使用) 打开 Navigation Bar，利用 `Alt + Insert` 快捷键新建文件或者文件夹。新建文件夹可以 `abc/def/ghi` 这样创建，新建 Java class 可以和包名一起，例如 `com.example.test.Test`。

* Multiple cursors

    多光标详见 [Introduces Sublime Text Style Multiple Selections](https://blog.jetbrains.com/idea/2014/03/intellij-idea-13-1-rc-introduces-sublime-text-style-multiple-selections/)。

* Language injection

    添加 JSON 或者 正则表达式等的时候，直接键入很麻烦，可以利用 Language injection 这个功能。以 JSON 为例，在双引号中 `Alt + Enter` 选择 Inject language -> JSON，然后 `Alt + Enter` 选择 Edit JSON Fragment，从新建窗口输入 JSON 字符串， 即可完成自动转换。

```java
String string = "";
// 输入{"name": "Tom"}
String string = "{\"name\": \"Tom\"}";
```

## Keymap

下面是一些快捷键组合，加粗的是我认为比较常用的快捷键。可以通过 Help -> Keymap Reference 查找快捷键（Windows 下 `Default` 的 Keymap）的 PDF 文档。

### Navigation

| Keymap(Win) | Action |
| ------------- | ------------- |
| **Ctrl + N** | Go to class |
| **Ctrl + Shift + N** | Go to file |
| **Ctrl + Shift + A** | Find Action |
| Ctrl + Alt + Shift + N | Go to symbol |
| **Ctrl + Alt + Left/Right**  | Navigate back/forward |
| Alt + Home | Show navigation bar |
| **Ctrl + E** | Recent files popup |
| Ctrl + Shift + E | Recnet edited files popup |
| Ctrl + F12 | File structure popup |
| Ctrl + H | Type hierarchy |
| Ctrl + Shift + H | Method hierarchy |
| Ctrl + Alt + H | Call hierarchy |
| Ctrl + B or Ctrl + Click | Go to declaration |
| Ctrl + Alt + B | Go to implementation(s) |
| Ctrl + U | Go to super-method/super-class |

值得注意的是，`Ctrl + Shift + E` 可能会与输入法的快捷键冲突，建议禁用输入法快捷键或者在英文状态下操作。`Ctrl + Alt + Left/Right` 会与 Intel 核显、网易云音乐（写本文时正在听歌，哈哈哈）等的快捷键冲突，建议禁用它们的快捷键。

### Editing

| Keymap(Win) | Action |
| ------------- | ------------- |
| Ctrl + Space | Basic code completion |
| Ctrl + Shift + Space | Smart code completion |
| Ctrl + Shift + Enter | Complete statement |
| **Ctrl + P** | Parameter info |
| Ctrl + Q | Quick documentation lookup |
| **Alt + Insert** | Generate code... or new file |
| Ctrl + O | Override methods |
| Ctrl + I | Implement methods |
| **Ctrl + Alt + T** | Surround with... |
| **Ctrl + W** | Select successively increasing code blocks |
| Ctrl + Shift + W | Decrease current selection to previous state |
| **Alt + Enter** | Show intention antions and quick-fixes |
| **Ctrl + Alt + L** | Reformat code |
| **Ctrl + Alt + O** | Optimize imports |
| Ctrl + Alt + I | Auto-indent line(s) |
| **Ctrl + D** | Duplicate current line or selected block |
| **Ctrl + Y** | Delete line at caret |
| Ctrl + Shift + U | Toggle case for word at caret or selected block |
| **Ctrl + Shift + Up/Down** | Move Statement Up/Down |
| Alt + Shift + Up/Down | Move Line Up/Down |

如何使用快捷键新建文件，参考 [这里](https://stackoverflow.com/questions/2249494/how-do-i-create-a-new-class-in-intellij-without-using-the-mouse)。

### Search/Replace

| Keymap(Win) | Action |
| ------------- | ------------- |
| **Double Shift** | Search everywhere |
| Ctrl + F | Find |
| F3 | Find next |
| Shift + F3 | Find pervious |
| Ctrl + R | Replace |
| Ctrl + Shift + F | Find in path |
| Ctrl + Shift + R | Replace in path |
| Alt + F7 / Ctrl + F7 | Find usages / Find usages in file |
| **Ctrl + Alt + J** | Surround with Live Template |
| **Ctrl + J** | Insert Live Template |
| **Ctrl + Shift + F12** | Toggle maximizing editor |

### Compile and Run

| Keymap(Win) | Action |
| ------------- | ------------- |
| Ctrl + F9 | Build project |
| Ctrl + Shift + F9 | Rebuild |
| Alt + Shift + F10 | Select configuration and run |
| Alt + Shift + F9 | Select configuration and debug |
| **Shift + F10** | Run |
| **Shift + F9** | Debug |
| Ctrl + F2 | Stop process |

### Debugging

| Keymap(Win) | Action |
| ------------- | ------------- |
| F8 | Step over |
| F7 | Step into |
| Shift + F7 | Smart step into |
| Shift + F8 | Step out |
| Alt + F9 | Run to cursor |
| Alt + F8 | Evaluate expression |
| F9 | Resume program |
| Ctrl + F8 | Toggle breakpoint |
| Ctrl + Shift + F8 | View breakpoints |

### Refactoring

| Keymap(Win) | Action |
| ------------- | ------------- |
| F5 | Copy |
| F6 | Move |
| Alt + Delete | Safe Delete |
| **Shift + F6** | Rename |
| Ctrl + F6 | Change Signatrue |
| Ctrl + Alt + N | Inline |
| **Ctrl + Alt + M** | Extract Method |
| Ctrl + Alt + V | Extract Variable |
| Ctrl + Alt + F | Extract Field |
| Ctrl + Alt + C | Extract Constant |
| Ctrl + Alt + P | Extract Parameter |

### General

| Keymap(Win) | Action |
| ------------- | ------------- |
| **Ctrl + Alt + S** | Open Setting dialog |
| **Ctrl + Alt + Shift + S** | Open Project Structure dialog |
| **Alt + 1** | Activate Project window |
| Alt + F12 | Activate Terminal window |

## 推荐阅读

* [42 IntelliJ IDEA Tips and Tricks](https://www.youtube.com/watch?v=eq3KiAH4IBI)

* [IntelliJ IDEA Tips and Tricks](https://blog.jetbrains.com/idea/2015/10/intellij-idea-tips-and-tricks/)

* [How do i create a new class in IntelliJ without using the mouse?](https://stackoverflow.com/questions/2249494/how-do-i-create-a-new-class-in-intellij-without-using-the-mouse)

* [Introduces Sublime Text Style Multiple Selections](https://blog.jetbrains.com/idea/2014/03/intellij-idea-13-1-rc-introduces-sublime-text-style-multiple-selections/)
