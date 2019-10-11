---
title: How to use matplotlib
date: 2018-05-07 14:27:06
tags: Matplotlib
categories: Python
---

{% blockquote Matplotlib https://matplotlib.org/ Matplotlib 2.2.2 documentation %}
Matplotlib is a Python 2D plotting library which produces publication quality figures in a variety of hardcopy formats and interactive environments across platforms. Matplotlib can be used in Python scripts, the Python and IPython shells, the Jupyter notebook, web application servers, and four graphical user interface toolkits.
{% endblockquote %}

## Matplotlib介绍

Matplotlib 是 python 的绘图库。一副完整的图形(figure)包括标题(title)、坐标轴标题(X&Y axis label)、边框线(spines)、主要刻度线(major tick)、主要刻度线标题(major tick label)、次要刻度线(minor tick)、次要刻度线标题(minor tick label)、图例(legend)、网格线(grid)、线(line)以及标记(marker)等，如下图所示。

![](https://matplotlib.org/_images/anatomy.png)

<!-- more -->

## 简单例子

一个简单的例子，使用 pandas 从 Excel 中读取数据，画折线图。值得注意的是，安装完 pandas 需要 `pip install xlrd`，否则会报错：ImportError: No module named 'xlrd'。

```python
# -*- coding: utf-8 -*-
import pandas as pd
import matplotlib.pyplot as plt


# Import data
excel_path = 'data.xlsx'
data = pd.read_excel(excel_path, sheet_name='Sheet1')
# data.x是pandas.Series对象，可用tolist()方法转化为list
x = data.x.tolist()
y = data.y.tolist()

# Plot
plt.plot(x, y)
plt.show()
```
{% qnimg matplotlib/simple.png  %}

## 复杂例子

### 折线图

上个例子比较简单，matplotlib 提供了许多 API 来自定义 figure 的各种设置。

调整边框线的粗细需要同时调整图例和刻度线，即：

```python
width = 1
rcParams['axes.linewidth'] = width
legend.get_frame().set_linewidth(width)

ax.xaxis.set_tick_params(width=width)
ax.yaxis.set_tick_params(width=width)
```

```python
# -*- coding: utf-8 -*-
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
from matplotlib import rcParams

from util import cm2inch

width = 0.5
fontsize = 10.5

rcParams['axes.linewidth'] = width

# Import data
excel_path = 'C:data.xlsx'
data = pd.read_excel(excel_path, sheet_name='Sheet1')
a = data.a.tolist()
y = data.y.tolist()
z = data.z.tolist()

# Plot
# Colors from seaborn-paper
colors = ['#4878CF', '#6ACC65', '#D65F5F', '#B47CC7', '#C4AD66', '#77BEDB']
# 设置图片大小(cm)
fig, ax = plt.subplots(figsize=cm2inch((12, 7.5)))

plt.plot(a, y, marker='o', c=colors[0], linewidth=1, markersize=4, label='first')
plt.plot(a, z, marker='^', c=colors[1], linewidth=1, markersize=4, label='second')

plt.title('title')
plt.xlabel('hello world', fontsize=fontsize)
plt.ylabel('hello world', fontsize=fontsize)
# 刻度线标记
plt.xlim(1, 12)
plt.xticks(np.arange(1, 13, 1))
plt.ylim((80, 220))
plt.yticks(np.arange(80, 221, 20))

legend = plt.legend(loc='best', edgecolor='inherit', fontsize=fontsize)
legend.get_frame().set_linewidth(width)

# Set the font size for axis tick labels
for tick in ax.get_xticklabels():
    tick.set_fontsize(fontsize)
for tick in ax.get_yticklabels():
    tick.set_fontsize(fontsize)

ax.xaxis.set_tick_params(width=width)
ax.yaxis.set_tick_params(width=width)
# 内侧刻度线
plt.tick_params(direction='in')

# 调整整体空白
fig.tight_layout()
# 保存图片
plt.savefig('fig.png', dpi=600)
plt.show()
```
{% qnimg matplotlib/plot.png  %}

### 柱状图

柱状图相对复杂些，需要设置每个柱体的宽度。

另外，可以设置每个主要刻度线的标记(tick)。

```python
# -*- coding: utf-8 -*-
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np

# Import data
excel_path = 'C:data.xlsx'
data = pd.read_excel(excel_path, sheet_name='Sheet1')
# 支持+ -运算
a = np.arange(1, len(data.a) + 1, 1)
ticks= ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
b = data.b.tolist()
c = data.c.tolist()
d = data.d.tolist()

# Plot
# Colors from seaborn-paper
colors = ['#4878CF', '#6ACC65', '#D65F5F', '#B47CC7', '#C4AD66', '#77BEDB']
fig, ax = plt.subplots()

total_width, n = 0.9, 3
width = total_width / n
a = a - (total_width - width) / 2

plt.bar(a, b,  width=width, color=colors[0], label='first')
plt.bar(a + width, c, width=width, color=colors[1], label='second')
plt.bar(a + 2 * width, d, width=width, color=colors[2], label='third')
plt.xticks(a + width, ticks)

legend = plt.legend(loc='best', edgecolor='inherit')

# 调整整体空白
fig.tight_layout()
# 保存图片
plt.savefig('bar.png', dpi=600)
plt.show()
```

{% qnimg matplotlib/bar.png  %}

## subplot

使用 matplotlib 可以很方便的绘制子图。`abc` 代表 a 行 b 列第 c 个图，顺序从左到右，从上往下。

```python
# a为横坐标值，b、c、d、e为纵坐标值
plt.subplot(221)
plt.plot(a, b)

plt.subplot(222)
plt.plot(a, c)

plt.subplot(223)
plt.plot(a, d)

plt.subplot(224)
plt.plot(a, e)
plt.show()
```

{% qnimg matplotlib/four.png  %}

当第一行有两个子图，第二行只有一个的时候，需要使用 [GridSpec](https://matplotlib.org/users/gridspec.html)。

```python
gs = gridspec.GridSpec(2, 4)
gs.update(wspace=0.5)
ax1 = plt.subplot(gs[0, :2], )
ax1.plot(a, b)

ax2 = plt.subplot(gs[0, 2:])
ax2.plot(a, c)

ax3 = plt.subplot(gs[1, 1:3])
ax3.plot(a, d)
plt.show()
```

{% qnimg matplotlib/three.png  %}


## 双坐标轴

使用 twinx() 方法，共享横坐标轴。

```python
fig, ax1 = plt.subplots()
lns1 = ax1.plot(a, b, 'b-', label='label1')
ax1.set_xlabel('xlabel')
# Make the y-axis label, ticks and tick labels match the line color.
ax1.set_ylabel('ylabel1', color='b')
ax1.tick_params('y', colors='b')

ax2 = ax1.twinx()
lns2 = ax2.plot(a, z, 'r:', label='label2')
ax2.set_ylabel('ylabel2', color='r')
ax2.tick_params('y', colors='r')

# added these three lines
lns = lns1+lns2
labs = [l.get_label() for l in lns]
legend = ax1.legend(lns, labs, loc=2, edgecolor='inherit')

fig.tight_layout()
plt.show()
```

{% qnimg matplotlib/two_scales.png  %}

## 字体

默认情况下，matplotlib 不支持中文，解决办法参考知乎的[讨论](https://www.zhihu.com/question/25404709)。但对一些强迫症来说，中文要是宋体或黑体，英文必须是 Times New Roman。一种完美的解决方案是使用 LaTeX 渲染，不过图片生成的时间比较长。

使用 LaTeX  渲染，需要安装 LaTeX 和 Ghostscript，否则一直会报错：FileNotFoundError: [WinError 2] 系统找不到指定的文件。参见官网的使用指南 [Text rendering With LaTeX](https://matplotlib.org/users/usetex.html)。

Windows 版本的 Protext 可以从 [清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/windows/protext/)下载，但一定记住下载 `protext.exe` 这个文件，那个带版本号的文件有问题，安装不了。

```python
# -*- coding: utf-8 -*-
from __future__ import unicode_literals

import matplotlib.pyplot as plt
import pandas as pd
from matplotlib import rcParams


rcParams['font.size'] = 14
# 使用LaTeX编译，CJK宏包，utf-8编码格式，宋体gbsn，黑体hei，楷体gai
rcParams['text.usetex'] = True
rcParams['text.latex.unicode'] = True
rcParams['text.latex.preamble'] = [
    r'\usepackage{CJK}',
    r'\AtBeginDocument{\begin{CJK}{UTF8}{song}}',
    r'\usepackage{mathptmx}',
    r'\AtEndDocument{\end{CJK}}'
]

# Import data
excel_path = 'data.xlsx'
data = pd.read_excel(excel_path, sheet_name='Sheet1')
a = data.a.tolist()
b = data.b.tolist()
c = data.c.tolist()

# Plot
# Colors from seaborn-paper
colors = ['#4878CF', '#6ACC65', '#D65F5F', '#B47CC7', '#C4AD66', '#77BEDB']
fig, ax = plt.subplots()

plt.scatter(a, b, label=r'\textrm{MOEA/D}')
plt.scatter(a, c, label=r'\textrm{NSGA-}\uppercase\expandafter{\romannumeral2}')

plt.xlabel(r'\textrm{hello world(kWh)}')
plt.ylabel(r'\textrm{这是中文哈哈哈($m^3/s$)}')

legend = plt.legend(loc='best', edgecolor='inherit')

# 内侧刻度线
plt.tick_params(direction='in')

# 调整整体空白
fig.tight_layout()
fig.subplots_adjust(bottom=0.1)

plt.savefig('latex.png', dpi=600)
plt.show()
```

{% qnimg matplotlib/latex.png  %}


Perfect！

## 推荐阅读

* [Matplotlib: Python plotting  &mdash; Matplotlib 2.2.2 documentation](https://matplotlib.org/)

* [用 matplotlib 画出规范的论文插图](http://www.seekiu.com/2016/mpl-for-publish.html)

* [Changing fonts in matplotlib](http://jonathansoma.com/lede/data-studio/matplotlib/changing-fonts-in-matplotlib/)
