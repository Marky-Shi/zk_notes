## Lattice 算法

格子密码（**Lattice-based Cryptography**）是在结构本身或在安全证明中涉及格 (Lattice)的加密图元构造的通用术语。基于晶格的构造目前是后量子密码学的重要候选者。

基本概念是格(lattice),由一组向量的线性组合的集合。这些向量的系数为整数。

> $$
> v_1 v_2 v_3 ...... v_n
> $$
>
> 在n维欧几里得空间 
> $$
> R^n
> $$
> 中的独立向量，格记住L，它是由
> $$
>  {v_1,v_2,v_3...... v_n}
> $$
> 的线性组合，其中 v1...... Vn 为整数。



格子密码的困难问题包括：

* 最短向量问题 (Shortest Vector Problem, SVP)：给定基向量，找到那个最短的非0向量。
* 最近向量问题 (Closest Vector Problem, CVP)：对于空间上任意一点（不一定要在lattice上），找到一个lattice点离他最近。

优点：

* 格子密码是平均困难（Average-Case Hardness）而不是 Worst-Case Hardness。
* 格子密码天然具有抗量子攻击的特性。
* 格子密码本质上非常简单就是基向量和其组成的离散点，非常简单易用，应用场景也非常广泛。

格的**秩(\**rank of lattice)为n，格的\**维度**(dimension)为m。如果 *m*=*n* ，那么则称这个格为**满秩格**(full-rank lattice)