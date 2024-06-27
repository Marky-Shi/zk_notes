### secp256k1 签名算法

secp256k1 是一种椭圆曲线，主要用于比特币和其他加密货币。它基于 Koblitz 曲线，定义如下：

- 曲线方程：
  $$
  y^2=x^3+7y^2=x^3+7
  $$
  
- 素数域：
  $$
  p=2^{256} −2^{32} −977
  $$
  
- 基点：
  $$
  G=(x,y)
  $$
   (具体坐标值已固定)

- 阶：
  $$
  n
  $$
  

#### 签名生成步骤（ECDSA）

1. **密钥生成**:

   - 生成一个随机私钥 d，在范围 [1,n−1][1,n−1] 内。

   - 计算公钥 
     $$
     Q=dG
     $$
     。

   ```
   Private key: d (random integer in range [1, n-1])
   Public key: Q = dG
   ```

2. **签名生成**:

   - 选择一个随机数 kk，在范围 [1,n−1][1,n−1] 内。

   - 计算 
     $$
     R=kG
     $$
     ，取 R 的 x 坐标作为 r。

   - 计算 
     $$
     e=HASH(m)
     $$
     。

   - 计算 
     $$
     s=k^−1(e+dr)mod  n
     $$
     。

   - 签名为 
     $$
     (r,s)
     $$
     。

   ```shell
   k (random integer in range [1, n-1])
   R = kG
   r = x-coordinate of R
   e = HASH(message)
   s = k^{-1}(e + dr) mod n
   Signature: (r, s)
   ```

3. **签名验证**:

   - 计算 
     $$
     e=HASH(m)
     $$
     。

   - 计算
     $$
     u_1=es^{−1} mod  n  与  u_2=rs−^1 mod  n
     $$
     
   - 计算
     $$
     R′=u_1G+u_2Q
     $$
     。
   
   - 验证 R′ 的 x 坐标是否等于 r。
   
   ```
   e = HASH(message)
   u1 = e s^{-1} mod n
   u2 = r s^{-1} mod n
   R' = u1G + u2Q
   Verify that the x-coordinate of R' equals r
   ```

### secp256r1 签名算法

secp256r1 (也称为 P-256 或 prime256v1) 是另一种椭圆曲线，被广泛应用于 TLS/SSL 和其他加密标准。它基于标准素数域曲线，定义如下：

- 曲线方程：
  $$
  y^2=x^3−3x+b
  $$
  （其中 b 是常数）
- 素数域：
  $$
  p = 2^{256} - 2^{224} + 2^{192} + 2^{96} - 1
  $$
  
- 基点：
  $$
  G=(x,y)
  $$
  (具体坐标值已固定)
- 阶：n

#### 签名生成步骤（ECDSA）

1. **密钥生成**:

   - 生成一个随机私钥 dd，在范围 [1,n−1][1,n−1] 内。
   - 计算公钥 
     $$
     Q=dG
     $$
     。

   ```
   Private key: d (random integer in range [1, n-1])
   Public key: Q = dG
   ```

2. **签名生成**:

   - 选择一个随机数 kk，在范围 [1,n−1][1,n−1] 内。
   - 计算 
     $$
     R=kG
     $$
     ，取 R 的 x 坐标作为 r。
   - 计算
     $$
     e=HASH(m)
     $$
     。
   - 计算 
     $$
     s=k^{−1} (e+dr)mod  n
     $$
     。
   - 签名为 (r,s)(r,s)。

   ```
   k (random integer in range [1, n-1])
   R = kG
   r = x-coordinate of R
   e = HASH(message)
   s = k^{-1}(e + dr) mod n
   Signature: (r, s)
   ```

3. **签名验证**:

   - 计算
     $$
     e=HASH(m)
     $$
     )。
   - 计算 
     $$
     u_1=es^{−1}mod  n 和 u_2=rs^{−1}mod  n
     $$
     
   - 计算
     $$
     R′=u_1G+u_2Q
     $$
     
   - 验证 R′的 x 坐标是否等于 r。
   
   ```
   e = HASH(message)
   u1 = e s^{-1} mod n
   u2 = r s^{-1} mod n
   R' = u1G + u2Q
   Verify that the x-coordinate of R' equals r
   ```

### 比较

- **secp256k1**:
  - 应用：主要用于比特币和其他加密货币。
  - 特性：基于 Koblitz 曲线，计算效率高。
- **secp256r1**:
  - 应用：广泛用于 TLS/SSL、智能卡等加密标准。
  - 特性：基于标准素数域曲线，广泛接受和标准化。

两种曲线在安全性上都非常强大，选择取决于具体的应用场景和标准要求。