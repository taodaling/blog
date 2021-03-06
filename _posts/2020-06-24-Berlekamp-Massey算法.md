---
categories: algorithm
layout: post
---

- Table
{:toc}

# Berlekamp-Massey算法

要了解Berlekamp-Massey算法的原理，可以看一下这篇[论文](https://pdfs.semanticscholar.org/a087/0e57fb5e59aae83f2a2a09b8df78257ef556.pdf)。

Berlekamp-Massey算法可以用于寻找线性递推数列的最短递推关系。所谓的线性递推数列是指这样的一个无穷数列$a_0,a_1,\ldots$，而线性递推关系是指一组系数$c_0,\ldots,c_{k-1}$，$k$为它的长度，满足对任意$i\geq k$，都有$a_k=\sum_{i=0}^{k-1}c_ia_{k-i-1}$。

实际上，如果我们记$x^i=a_i$，则可以发现最短递推关系实际上是一个最小多项式：

$$
\begin{aligned}
Z_a(x)&=x^k-\sum_{i=0}^{k-1}c_ix^{k-i-1}
\end{aligned}
$$

## BM算法求解递最短线性递推关系

如果已知递推关系长度不超过$n$，那么我们需要递推数列的前面前面多少项才能计算出正确的递推关系，记这个数为$m$。BM算法的论文中指出之后如果由于最短递推关系不再满足而发生重构，那么重构后的递推关系不小于$\lceil \frac{m+1}{2} \rceil$，但是我们提过整个序列的最短递推关系不会超出$n$项，因此我们选择$m=2n$就足够了。

下面再讨论一种情况，在模$p$的意义下，要求我们为一个矩阵$n\times m$的矩阵组成的序列$A_1,\ldots,A_k$计算最短线性递推关系。我们可以随机一个$n$维的列向量$u$和$m$维的行向量$v$，之后找到序列$uA_1v,uA_2v,\ldots,uA_kv$的最短线性递推关系，可以通过Schwartz–Zippel定理定理推出$A_i$和$uA_iv$两个序列的最短线性递推关系不同的概率仅为$\frac{n+m}{p}$。

## BM算法求解递推数列第$n$项

我们要计算的是$a_n$，由于$a_n=x^n$，而$Z_a(x)$是它的零化多项式，因此我们可以通过多项式取模得到$g(x)=x^n\pmod{Z_a(x)}$，得到的结果$g(x)$的阶数小于$k$，因此可以直接求解。

取模的时间复杂度为$O(k^2\log_2n)$，如果使用FFT则可以优化到$O(k\log_2k\log_2n)$。

## BM算法求解矩阵$A$的最小多项式

$A$是$n\times n$矩阵，且只有$m$个元素非$0$。设$poly(A)$为其最小多项式。

考虑这样的向量数列$A^0v,A^1v,A^2v\ldots$。可以发现$poly(A)$也是数列的零化多项式（令$x^i=A^iv$）。从这样的数列$A^0v,A^1v,A^2v\ldots$中利用BM求最小递推关系，时间复杂度为$O(n^3)$。

对于稀疏矩阵，我们这边可以使用一个混淆的方式，就是求数列$uA^0v,uA^1v,uA^2v\ldots$的最小递推关系，它的时间复杂度是$O(n(n+m))$。

上面提到的$u,v$都是随机选择的向量。

## BM算法求解$det(A)$

$A$是$n\times n$矩阵，且只有$m$个元素非$0$。设$poly(A)$为其最小多项式。

根据特征多项式的定义$f(t)=det(t-A)$，可知$det(A)=(-1)^nf(0)$。因此我们只需要求出矩阵$A$的特征多项式，求其常数项即可。

已知的求特征多项式的方式是将矩阵转换成上海森堡形式，之后通过多项式插值求解，时间复杂度为$O(n^3)$，和直接通过高斯消元求行列式时间复杂度相同。我们不能直接求解最小多项式，因为最小多项式不一定是特征多项式，不过二者的常数项均只有在矩阵$A$奇异的情况下才会为$0$。

对于稀疏矩阵，我们可以生成一个随机的对角线矩阵$D$，之后用BM算法计算$DA$的最小多项式。而$DA$的最小多项式有很大的概率就是$DA$的特征多项式，于是我们得到$poly(DA)$，从而得到$det(DA)$。而$det(A)=\frac{det(DA)}{det(D)}$，这是很好算的。这种方法的时间复杂度为$O(n(n+m))$。

## BM算法求解$A^{-1}b$

对于线性方程组$Ax=b$，如果$A$可逆，则可以通过$x=A^{-1}b$求解。（这种情况非常常见，比如求随机游走的概率和期望的时候）。

对于一般的密集矩阵，我们可以通过直接对$A$求逆计算结果，时间复杂度为$O(n^3)$。

对于稀疏矩阵。我们可以先计算出$A$的最小多项式$g(A)=\sum_{i=0}^{k}c_iA^i$。之后进行推导：

$$
\begin{aligned}
&\sum_{i=0}^{k}c_iA^i=0\\
\Rightarrow &-c_0=\sum_{i=1}^{k}c_iA^i\\
\Rightarrow &I=-\frac{1}{c_0}\sum_{i=1}^{k}c_iA^i\\
\Rightarrow &A^{-1}b=-\frac{1}{c_0}\sum_{i=0}^{k-1}c_{i+1}A^ib\\
\end{aligned}
$$

上面的过程我们可以$O(n(n+m))$求解。

# 参考资料

- [2019暑期金华集训 Day2 线性代数
自闭集训 Day2](https://www.cnblogs.com/p-b-p-b/p/11305229.html)
- 《两类递推数列的性质和应用》