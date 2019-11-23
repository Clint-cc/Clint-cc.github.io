---
layout: post
title: "数据分析之Matplotlib模快"
date: 2018-12-20
description: "Matplotlib模快学习笔记"
tag: 数据分析
---   
## 前言
这里主要记录下我在学习数据分析时所做的一些笔记，数据分析将会作为一门通识技能，进入越来越多的不同工作岗位中。毕竟“技多不压身”，掌握数据分析，一方面可以提升自己相应的业务能力，另一方面也可以让自己建立一种data-driven的视角，去思考各种问题。不多说，上笔记！<hr>
## 利用plt作折线图
### 一个简单的栗子
```python
# -*- coding:utf-8 -*-
# @Author  : Clint
from matplotlib import pyplot as plt

fig = plt.figure(figsize=(20, 8), dpi=80)  # 设置图片的大小及像素

x = range(2, 26, 2)
y = [15, 13, 14.5, 17, 20, 25, 26, 26, 27, 22, 18, 15]

# 绘图
plt.plot(x, y)

# 设置x轴的刻度
_xtick_labels = [i/2 for i in range(2, 49)]
plt.xticks(_xtick_labels)

# 设置y轴刻度
plt.yticks(range(min(y), max(y)+1))

# 保存图片
# plt.savefig("./sig_size.png")

# 展示
plt.show()
```
结果：
![avatar](/images/posts/data1.png)

### 更详细的参数栗子
```python
# -*- coding:utf-8 -*-
# @Author  : Clint

from matplotlib import pyplot as plt
plt.rcParams['font.sans-serif'] = ['SimHei']   # 解决中文显示问题

fig = plt.figure(figsize=(20, 8), dpi=80)  # 图形大小

x = range(11, 31)   # x轴坐标范围
y = [1, 0, 1, 1, 2, 4, 3, 2, 3, 4, 4, 5, 6, 5, 4, 3, 3, 1, 1, 1]
z = [1, 0, 3, 1, 2, 2, 3, 3, 2, 1, 2, 1, 1, 1, 4, 1, 3, 1, 1, 1]

plt.plot(x, y, 'b--', label='我')
plt.plot(x, z, color='red', label='同桌')

plt.xticks(x[::1], ['{}岁'.format(i) for i in x])
plt.yticks(y)

# 绘制网格
plt.grid(alpha=0.1)      # alpha: 网格透明度

plt.xlabel("岁数")   # x轴刻度说明
plt.ylabel("女(男)朋友个数")  # y轴刻度说明
plt.title("11到30岁每年交往朋友的数量趋势")  # 图片标题

# 添加图例
plt.legend()   # 参数: loc="upper left" 放在左上方

plt.show()     # 展示
```
![avatar](/images/posts/data2.png)
## 散点图
### 散点图和折线图的代码几乎相似,只是作图的函数不一样
```python
plt.scatter(x_3, y_3, label="3月份")      # 作图函数为scatter
plt.scatter(x_10, y_10, label="10月份")
```
### 其他的图形做法也都差不多，作图过程中我们应该不断的调整图的参数以达到最后我们想要的效果
### 当然还要很多类型的图，我们需要做的是更多的动手实践，实践出真知！

## 其他
关于作图，其实有很多的作图网站，网站中有很多的模板，比我们自己做的要好得多，而且是动态图，我们只需要将数据放入其中即可，下面放链接:

[百度的echart](https://echarts.baidu.com)<br>
[plotly官网](https://dash.plot.ly)
