---
title: LaTeX 数学公式渲染指南
date: 2025-07-15 10:00:00
updated: 2025-07-15 10:00:00
tags:
  - LaTeX
  - MathJax
  - 数学
  - 教程
categories:
  - 技术
keywords: latex, mathjax, 数学公式, hexo, butterfly
description: 在 Hexo Butterfly 博客中使用 MathJax 渲染 LaTeX 数学公式的完整指南。
top_img:
cover:
mathjax: true
comments: true
toc: true
---

## 概述

Butterfly 主题内置了 [MathJax](https://www.mathjax.org/) 支持，只需在文章 Front-matter 中设置 `mathjax: true` 即可启用 LaTeX 公式渲染。

## 行内公式

行内公式使用 `$...$` 包裹，例如：

- 欧拉公式：$e^{i\pi} + 1 = 0$
- 勾股定理：$a^2 + b^2 = c^2$
- 质能方程：$E = mc^2$
- 正态分布：$f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{(x-\mu)^2}{2\sigma^2}}$

## 块级公式

### 柯西-施瓦茨不等式

$$
\left( \sum_{k=1}^{n} a_k b_k \right)^2 \leq
\left( \sum_{k=1}^{n} a_k^2 \right)
\left( \sum_{k=1}^{n} b_k^2 \right)
$$

### 傅里叶变换

$$
\mathcal{F}\{f(t)\} = F(\omega) = \int_{-\infty}^{\infty} f(t) e^{-i\omega t} \, dt
$$

### 矩阵

$$
\mathbf{A} = \begin{bmatrix}
a_{11} & a_{12} & \cdots & a_{1n} \\
a_{21} & a_{22} & \cdots & a_{2n} \\
\vdots & \vdots & \ddots & \vdots \\
a_{m1} & a_{m2} & \cdots & a_{mn}
\end{bmatrix}
$$

### 贝叶斯定理

$$
P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}
$$

## 多行公式（麦克斯韦方程组）

$$
\begin{aligned}
\nabla \cdot \mathbf{E} &= \frac{\rho}{\varepsilon_0} \\
\nabla \cdot \mathbf{B} &= 0 \\
\nabla \times \mathbf{E} &= -\frac{\partial \mathbf{B}}{\partial t} \\
\nabla \times \mathbf{B} &= \mu_0 \left( \mathbf{J} + \varepsilon_0 \frac{\partial \mathbf{E}}{\partial t} \right)
\end{aligned}
$$

## 微积分

### 极限

$$
\lim_{n \to \infty} \left(1 + \frac{1}{n}\right)^n = e
$$

### 积分

$$
\int_{0}^{\pi} \sin x \, dx = 2
$$

## 机器学习常用公式

### 线性回归损失函数

$$
J(\theta) = \frac{1}{2m} \sum_{i=1}^{m} \left( h_\theta(x^{(i)}) - y^{(i)} \right)^2
$$

### Softmax 函数

$$
\sigma(\mathbf{z})_i = \frac{e^{z_i}}{\sum_{j=1}^{K} e^{z_j}}
$$

### 交叉熵损失

$$
\mathcal{L} = -\sum_{c=1}^{C} y_c \log(\hat{y}_c)
$$

---

> 更多 LaTeX 语法请参考 [LaTeX Wiki](https://en.wikibooks.org/wiki/LaTeX/Mathematics)。
