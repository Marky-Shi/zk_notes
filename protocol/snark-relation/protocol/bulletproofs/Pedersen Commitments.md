## Pedersen Commitments
一种密码学中承诺方案，具有同态加法的性质，且在信息论上是隐藏的。这意味着即使拥有无限的计算能力，也无法从承诺中获取被承诺的值的信息。Pedersen 承诺在密码学协议中被广泛使用，尤其是在零知识证明、安全多方计算和承诺方案中。

### 定义

幂运算版本（整数模P群）：
$$ [Com(m, r) = g^m \cdot h^r \pmod{p}] $$

假设我们有一个群 G，其阶数为一个大素数 q，生成元为 g 和 h。g 和 h 必须是独立生成的，也就是说，不存在一个已知的标量 x 使得 h = g<sup>x</sup>。这个关系通常被称为“离散对数问题”的困难性假设。

椭圆曲线版本（椭圆曲线群）：
$$ [Com(m, r) = m \cdot G + r \cdot H]$$

G和H是椭圆曲线上的两个生成元（或者一个循环群中的基）

### 验证过程：
承诺者在揭示时提供 m和 r ，验证者计算：

$$ [Com(m, r) \stackrel{?}{=} g^m \cdot h^r \pmod{p}]$$

或者
$$[Com\stackrel{?}{=} m \cdot G + r \cdot H]$$

### 性质
1. 隐藏性
    - 如果 r 是随机选择的，那么给定承诺 Com(x, r)，任何人都无法确定 x 的值，即使拥有无限的计算能力。这是因为对于任何给定的 x，总能找到一个 r' 使得 Com(x', r') = Com(x, r)，其中 x' ≠ x。由于 g 和 h 是独立生成的，r 的变化会使得承诺值在群 G 中均匀分布，从而隐藏了 x 的信息。

2. 绑定性
    - 承诺者无法找到一个 r' 使得 Com(x', r') = Com(x, r) 且 x' ≠ x。这是基于离散对数问题的困难性假设。如果承诺者能够找到这样的 r'，那么他/她就相当于解决了离散对数问题，即找到了一个 x 使得 h = g<sup>x</sup>，这在计算上是不可行的。

3. 加法同态性
    - 如果 x 和 y 是两个已知的值，那么 Com(x + y, r) = Com(x, r) * Com(y, r)。这允许 Pedersen 承诺在计算中实现加法运算，从而实现加密和签名。



例子： 如一个阶数为 17 的群 G，生成元 g = 3，h = 5。我们要承诺值 x = 7，并选择一个随机数 r = 11。那么承诺值为：
$$com = g^{x}h^{r}\pmod{p} = 3^{7}5^{11}\pmod{17} = 3^{7}5^{11} \bmod 17 = 11 * 11 \pmod {17} = 121 \pmod {17}$$



code example :
```python
from py_ecc.bn128 import is_on_curve, FQ
from py_ecc.bn128 import is_on_curve, FQ
from py_ecc.fields import field_properties
field_mod = field_properties["bn128"]["field_modulus"]
from hashlib import sha256
from libnum import has_sqrtmod_prime_power, sqrtmod_prime_power

b = 3 # for bn128, y^2 = x^3 + 3
seed = "commitments"

x = int(sha256(seed.encode('ascii')).hexdigest(), 16) % field_mod 

entropy = 0

vector_basis = []

#generate n points

n = 10
for _ in range(n):
    while not has_sqrtmod_prime_power((x**3 + b) % field_mod, field_mod, 1):
        # increment x, so hopefully we are on the curve
        x = (x + 1) % field_mod
        entropy = entropy + 1

    # pick the upper or lower point depending on if entropy is even or odd
    y = list(sqrtmod_prime_power((x**3 + b) % field_mod, field_mod, 1))[entropy & 1 == 0]
    point = (FQ(x), FQ(y))
    assert is_on_curve(point, b), "sanity check"
    vector_basis.append(point)

    # new x value
    x = int(sha256(str(x).encode('ascii')).hexdigest(), 16) % field_mod 
print(vector_basis)

```
