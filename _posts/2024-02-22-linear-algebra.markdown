---
layout: post
title:  "趙啟超 線性代數筆記"
date:   2024-02-22 23:00:00 +0800
categories: math
mathjax: true
---
### [Chapter 3] Vector Spaces and Subspaces

#### [3.1] Spaces of Vectors

- *(Definition)*  
	$$
	\begin{align}
	\mathbb{R}^n &= \text{all (column) vectors with n real components} \\\\
	&=\{\,\underline{v}=(v_1,v_2,\cdots,v_n)\;\text{for each}\;v_i \in R\,\}
	\end{align}
	$$

	- vector addition: \\(\text{If}\;\underline{v},\,\underline{w}\in\mathbb{R}^n\text{, then }
	  \underline{v}+\underline{w}\in\mathbb{R}^n\\)
	- scalar multiplication: \\(\text{If}\;\underline{v}\in\mathbb{R}^n,\,c\in R\\), then
	  \\(\underline{v}+\underline{w}\;\in\mathbb{R}^n\\)	  

- *(Definition)* Vector Space
	- \\(V: \text{a set of vectors}\\)	
	- two operations:
		- vector addition: \\(\,\underline{v}\in V,\,\underline{w}\in V \Rightarrow
		  \underline{v}+\underline{w}\in V\\)
		- scalar multiplication: \\(\,\underline{v}\in V \Rightarrow
		  c\underline{v}\in V\;\text{where $c$ is any scalar}\\)
	- eight rules: \\(\,\underline{u}\in V,\,\underline{v}\in V,\,\underline{w}\in V\\)
		1. \\(\,\underline{v}+\underline{w}=\underline{w}+\underline{v}\,\\)
		2. \\(\,\underline{u}+(\underline{v}+\underline{w})=
		   （\underline{u}+\underline{v})+\underline{w}\,\\)
		3. There is a unique "zero vector" \\(\,\underline{0}\\)
		   such that \\(\,\underline{v}+\underline{0}=\underline{v}\\)
		4. For each \\(\,\underline{v}\\), there is a unique vector \\(\,-\underline{v}\\) such that
		   \\(\,\underline{v}+(-\underline{v})=\underline{0}\\)
		5. \\(\,1\cdot\underline{v}=\underline{v}\\)
		6. \\(\,(c_1c_2)\underline{v}=c_1(c_2\underline{v})\\)
		7. \\(\,c(\underline{v}+\underline{w})=c\underline{v}+c\underline{w}\\)
		8. \\(\,(c_1+c_2)\underline{v}=c_1\underline{v}+c_2\underline{v}\\)
- \\(Z:\lbrace\underline{0}\rbrace\\) is the smallest vector space
- *(Remark 1)*  
  $$\begin{align}
  1\cdot\underline{v}=\underline{v}
  &\Rightarrow(0+1)\underline{v}=\underline{v} \\
  &\Rightarrow0\cdot\underline{v}+1\cdot\underline{v}=\underline{v} \\
  &\Rightarrow0\cdot\underline{v}+\underline{v}=\underline{v} \\
  &\Rightarrow0\cdot\underline{v}+\underline{v}+(-\underline{v})=\underline{v}+(-\underline{v}) \\
  &\Rightarrow0\cdot\underline{v}+(\underline{v}+(-\underline{v}))=\underline{0} \\
  &\Rightarrow0\cdot\underline{v}+\underline{0}=\underline{0} \\
  &\Rightarrow0\cdot\underline{v}=\underline{0} \\
  \end{align}
  $$
- *(Remark 2)*  
  $$\begin{align}
  0\cdot\underline{v}=\underline{0}
  &\Rightarrow(1+(-1))\underline{v}=\underline{0} \\
  &\Rightarrow1\cdot\underline{v}+(-1)\cdot\underline{v}=\underline{0} \\
  &\Rightarrow\underline{v}+(-1)\cdot\underline{v}=\underline{v}+(-\underline{v}) \\
  &\Rightarrow(-1)\cdot\underline{v}=-\underline{v}
  \end{align}
  $$
- *(Definition)* A subset \\(W\\) of a vector space \\(V\\) is a subspace if \\(W\\) itself is a vector space
- *(Definition)* A subset \\(W\\) of a vector space \\(V\\) is a subspace if 
	1. \\(\,\underline{v}\in W,\,\underline{w}\in W \Rightarrow
	   \underline{v}+\underline{w}\in W\\)
	2. \\(\,\underline{v}\in W \Rightarrow
	   c\underline{v}\in W\;\text{for any scalar $c$}\\)
- Note: 對vector space \\(V\\), eight rules成立, 但必須從two operations證明對\\(W\\)也是成立.
  特別是rule(3)和rule(4)
  $$
  \text{Proof of rule(3): According to Remark 1} \\
  0\cdot\underline{v}=\underline{0} \\
  \Rightarrow\,\because0\cdot\underline{v}\in W\therefore\,\underline{0}\in W \\
  \text{(Claim)Every subspace contains the zero vector} \\
  \text{Proof of rule(4): According to Remark 2} \\
  (-1)\cdot\underline{v}=-\underline{v} \\
  \Rightarrow\,\because(-1)\cdot\underline{v}\in W\therefore\,-\underline{v}\in W
  $$
- *(Definition)*    
  The column space \\(C(A)\\) of a matrix \\(A\\):
  all linear combinations of the column vectors of \\(A\\)  
  \\(\therefore\text{the system }A\underline{x}=\underline{b}\text{ is solvable}\iff\underline{b}\in C(A)\\)
- *(Claim)*    
  If A is an \\(m \times n\\) real matrix, then \\(C(A)\\) is a subspace of \\(\mathbb{R}^m\\)  
  Proof: \\(A\\)的column vectors都在\\(\mathbb{R}^m\\)內, \\(C(A)\\)是\\(A\\)的column vectors產生的線性組合  
  的集合, 所以\\(C(A)\\)裡的vectors經過線性組合後仍然是\\(A\\)的column vectors產生的線性組合  
  故\\(C(A)\\)是\\(\mathbb{R}^m\\)的subspace
- *(General Case)*  
  \\(S\\): a set of vectors in a vector space \\(V\\) (S is probably not a subspace)  
  \\(SS\\): the set of all linear combinations of vectors in \\(S\\) (\\(SS\\) is the "span" of \\(S\\))  
  Then \\(SS\\) is a subspace of \\(V\\) (the subspace "spanned" by \\(S\\))  
  Note: If \\(S=\\) the set of column vectors of \\(A\\), then \\(SS=C(A)\\)