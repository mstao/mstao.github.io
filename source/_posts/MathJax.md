---
title: MathJax渲染Latex语法基本使用
categories: MathJax
tags: [MathJax]
author: Mingshan
date: 2019-1-15
mathjax: true
---

用Markdown写博客的时候，有时需要用到[Latex](https://www.latex-project.org/)来写数学公式，通过使用[MathJax](https://www.mathjax.org)，我们可以让Markdown解析Latex数学表达式，同时Next主题也支持MathJax，所以了解一下Latex语法是十分有必要的。

<!-- more -->

## 基础语法

### 公式标记

MathJax支持行内公式（inline）和陈列公式（displayed）。inline表示公式嵌入到文本段中，displayed表示公式独自成为一个段落。例如$f(n)=2n^2+3n+1$就是一个行内公式，而下面的则是displayed公式：

$$f(n)=2n^2+3n+1$$

在MathJax中，displayed公式分隔符为 `$$...$$` ，inline公式分隔符为`$...$`。

### 希腊字母

希腊字母在数学表达式中使用频率非常高，分为大小和小写，列表如下：

名称 |   大写  | Tex |  小写 | Tex
---|---|---|---|---
alpha    |    A   |     A     |     α    |   \alpha
beta     |    B   |     B     |     β    |   \beta
gamma    |    Γ   |   \Gamma  |     γ    |   \gamma
delta    |    Δ   |   \Delta  |     δ    |   \delta
epsilon  |    E   |     E     |     ϵ    |   \epsilon
zeta     |    Z   |     Z     |     ζ    |   \zeta
eta      |    H   |     H     |     η    |   \eta
theta    |    Θ   |   \Theta  |     θ    |   \theta
iota     |    I   |     I     |     ι    |   \iota
kappa    |    K   |     K     |     κ    |   \kappa
lambda   |    Λ   |  \Lambda  |     λ    |   \lambda
mu       |    M   |     M     |     μ    |   \mu
nu       |    N   |     N     |     ν    |   \nu
xi       |    Ξ   |    \Xi    |     ξ    |   \xi
omicron  |    O   |     O     |     ο    |   \omicron
pi       |    Π   |    \Pi    |     π    |   \pi
rho      |    P   |     P     |     ρ    |   \rho
sigma    |    Σ   |   \Sigma  |     σ    |   \sigma
tau      |    T   |     T     |     τ    |   \tau
upsilon  |    Υ   |  \Upsilon |     υ    |   \upsilon
phi      |    Φ   |    \Phi   |     ϕ    |   \phi
chi      |    X   |     X     |     χ    |   \chi
psi      |    Ψ   |    \Psi   |     ψ    |   \psi
omega    |    Ω   |   \Omega  |     ω    | \omega 


### 上下标和分组
指数和下标可以用`^`和`_`后加相应字符来实现，例如`x_i^2`: $x_i^2$。如果要实现分组效果，用`{}`来实现，使用{}将具有相同等级的内容放入其中，成组处理。例如`x^10`显示为：$x^10$，不是我们要的效果，`x^{10}`：$x^{10}$。

### 分式和根式

分式的表示：

- 第一种，使用`\frac ab` , \frac作用于其后的两个组a , b ，结果为ab。如果你的分子或分母不是单个字符，请使用{...}来分组。
- 第二种，使用`\over`来分隔一个组的前后两部分，如 `{a+1 \over b+1}`: ${a+1 \over b+1}$

根式使用`\sqrt`表示，如：`\sqrt[4]{\frac xy} `: $\sqrt[4]{\frac xy}$

### 求和、极限与积分

- 求和：`\sum`,其下标表示求和下限，上标表示上限。 例如`\sum_{i=1}^n{a_i}`，显示为：$\sum_{i=1}^n{a_i}$
- 极限：`\lim`, 例如`\lim_{x \to 0}`，会显示为：$\lim{x \to 0}$；又例如```\lim \limits_{x \to 1} \frac{x^2-1}{x-1}```，显示为：$\lim \limits_{x \to 1} \frac{x^2-1}{x-1}$
- 积分:  `\int`, 其上下标表示积分的上下限。例如`\int_0^\infty{fxdx}`,显示为：$\int_0^\infty{fxdx}$

### 矩阵

**表示格式**

矩阵的表示格式为：`\begin{matrix}…\end{matrix}`， 以`\begin`表示矩阵开始，以`\end`表示矩阵结束， 矩阵的每一行以`\\`结束, 矩阵的每一个元素之间以`&`分隔。下面是一个示例，语法如下：

    \begin{matrix}
    1 & x & x^2 \\
    1 & y & y^2 \\
    1 & z & z^2 \\
    \end{matrix}

效果为：

$$
    \begin{matrix}
    1 & x & x^2 \\
    1 & y & y^2 \\
    1 & z & z^2 \\
    \end{matrix}
$$

**括号种类**

我们知道矩阵外面的括号有很多种类，主要有以下几种：

`pmatrix`
$$
    \begin{pmatrix}
    1 & x \\
    1 & y \\
    1 & z \\
    \end{pmatrix}
$$

`bmatrix`
$$
    \begin{bmatrix}
    1 & x \\
    1 & y \\
    1 & z \\
    \end{bmatrix}
$$

`Bmatrix`
$$
    \begin{Bmatrix}
    1 & x \\
    1 & y \\
    1 & z \\
    \end{Bmatrix}
$$

`vmatrix`
$$
    \begin{vmatrix}
    1 & x \\
    1 & y \\
    1 & z \\
    \end{vmatrix}
$$

`Vmatrix`
$$
    \begin{Vmatrix}
    1 & x \\
    1 & y \\
    1 & z \\
    \end{Vmatrix}
$$


### 常用符号

- `\lt \gt \le \leq \leqq \leqslant \ge \geq \geqq \geqslant \neq \not\lt` $\lt \gt \le \leq \leqq \leqslant \ge \geq \geqq \geqslant \neq \not\lt$
- `\times \div \pm \mp` $\times \div \pm \mp$ `\cdot ` $x⋅y$
- `\cup \cap \setminus \subset \subseteq \subsetneq \supset \in \notin \emptyset \varnothing`  $\cup \cap \setminus \subset \subseteq \subsetneq \supset \in \notin \emptyset \varnothing$
- `{n+1 \choose 2k}` or `\binom{n+1}{2k} (n+12k)` ${n+1 \choose 2k}$
- `\to \rightarrow \leftarrow \Rightarrow \Leftarrow \mapsto` $\to \rightarrow \leftarrow \Rightarrow \Leftarrow \mapsto$
- `\land \lor \lnot \forall \exists \top \bot \vdash \vDash` $\land \lor \lnot \forall \exists \top \bot \vdash \vDash$
- `\star \ast \oplus \circ \bullet` $\star \ast \oplus \circ \bullet$
- `\approx \sim \simeq \cong \equiv \prec \lhd \therefore` $\approx \sim \simeq \cong \equiv \prec \lhd \therefore$
- `\infty \aleph_0 \nabla \partial \Im \Re` $\infty \aleph_0 \nabla \partial \Im \Re$
- `\ldots \cdots \ddots \vdots` $\ldots \cdots \ddots \vdots$


还有其他许多复杂的符号，我就不贴到这个地方了。详细的语法请参考下方的链接，你可以通过在线渲染的[地址](https://www.mathjax.org/#demo)来验证你写的数学公式是否正确。

## References：

- [MathJax basic tutorial and quick reference](https://math.meta.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference)
- [Mathjax Live Demo](https://www.mathjax.org/#demo)
- [Mathjax与LaTex公式简介](https://www.cnblogs.com/linxd/p/4955530.html)
- [常用数学符号的 LaTeX 表示方法](http://www.mohu.org/info/symbols/symbols.htm)
- [一份不太简短的 LATEX2e 介绍](http://www.mohu.org/info/lshort-cn.pdf)
