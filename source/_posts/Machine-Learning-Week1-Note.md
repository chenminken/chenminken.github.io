---
title: Machine Learning Week1 Note
date: 2019-09-13 23:02:24
tags: MachineLearning, Coursera
---

此系列博文是学习Coursera上Stanford的《Machine Learning》MOOC的笔记。

课程共分为11周，大约需要56小时学习。

此文是第一周的笔记。

## 学习动机

本课程是在同学的推荐下学习的，因为大四的两个项目都需要用到机器学习的知识和工具，所以需要先对机器学习有一个大概的了解。这门课程是很有名的。且Coursera上的评分为4.9，有250多万人注册过这门课！

Ng的课程说明中对本门课程的定义为不仅

##介绍

机器学习的应用有：

1. 垃圾邮件拦截。
2. 网页搜索。
3. 电子相册的面孔识别

机器学习适用的场景：代码不能手动编写出来。比如：

1. 直升机自动驾驶
2. 自然语言识别
3. 视觉识别

机器学习在数据挖掘中的应用。

推荐系统，点击流。

## 什么是机器学习？

公认的机器学习的定义有两种说法。一种是Mitchell提出的ETP定义，一种是

Arthur Samuel提出的学科定义。

### ETP（Experience, Task, Performance）

> A computer is said to learn from experience E with respect to some class of tasks T and performance measure P, if its performance at tasks in T, as measured by P, improves with experience E.

Experience，实验经历。

Task：任务。

Performance：预测的准确度。

### Samuel的学科定义

> Machine Learning is the field of study gives computers that ability to learn without being explicitly programmed.

机器学习是研究赋予不用显性的编写代码就能使计算机获得学习的能力的学科。

## 监督学习

监督学习是指输入训练特征数据，并且告诉你正确的答案是什么，然后再输入未知的特征，通过对之前的数据挖掘出规律预测未知输入对应的输出。

监督学习需要使用标记好的特征数据训练。

监督学习可分为两种形式。一种是分类（Classification），分类的结果是离散有限的。另外一种是回归（regression），回归的输出结果是连续的。以卖房为例，分类的结果可以是预测指定房子在未来三个月是否能卖出去。而回归的结果可以是符合指定条件（比如面积和地段）的房子的预期价格。价格既可以是10000，也可以是10001.23。

在上面的例子中，面积和地段是学习模型的两个特征，除了它们以外，可能的特征还有小区绿化率、1公里内的地铁站的数量等等特征。如果需要，甚至可以使用近乎无穷数量的特征进行监督学习。

## 非监督学习

非监督学习不需要知道输入数据的标签和分类，而是通过输入数据的结构聚类（Clustering）。比如google新闻推荐，自动以一个新闻事件为主题，聚集相关的新闻报道。或者社交网络中人群分类都可以使用非监督学习进行。

课程中举了一个Cokcktail Party Problem的问题。即通过两个不同位置的声源分离出环境音和人声。

**TODO：实现一个算法**

