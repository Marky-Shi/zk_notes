## Halo2 Q & A

1. **`assign_advice_from_constant`**  **`assign_advice`** 的区别： 

   1. `assign_adive_from_constant` 
      1. 在keygen的时候，为fix 列分配一个常量。
      2. 在keygen的时候， 在fix 和advice 设置相同的约束。
      3. 在proof gen 的时候，将相同的值分配给advice cell
   2. advice 只会在生成proof的时候，讲值分配给advice cell。

2. PSE halo2 与原halo2的区别

   1. `commitment schemes`: 原Halo2 只支持IPA，PSE 中还支持KGZ
   2. `lookup`:  原支持固定查表，PSE中支持动态查表
      1. `meta.lookup_any` PSE中提供这个接口，可以进行动态查表。
   3. `Challenge API` PSE 中支持多轮承诺和挑战

3. What is the difference between public input and the constant?

   1. PI 可以在证明者的不同调用之间发生变化，而常数在所有证明中都是固定的。

4. circuit layouts

   1. 在halo2生成的电路图中： 红色代表 advice column，蓝色为fix， 白色部分则是instance/public inputs，绿色则是region的调用部分。

5. 在halo2 中，每个门都应用于每个row，将gate想象为俄罗斯方块，将其放在每一行上，而俄罗斯方块覆盖的每个单元格都由该行上的门使用。这就是为什么有`selector`，可以使用它来仅在特定行上启用门的效果。

6. `copy_advice` 和 `assign_advice_from_instance` 区别

   1. `copy_advice` 在特定偏移处的两列之间添加了 `copy_constraint`，**同时将值从一列设置为另一列** 。这不仅提供了`witness`，而且还强制排列参数包含复制约束。与 `assign_advice_from_instance` 的具体区别在于，`copy_advice` 复制 `Advice` 列，而其他复制从 `Instance` 列（pulic inputs）到 `Advice` 列。

   2. `assign_advice_from_instance -> Copy From Instance -> Advice`
   3. `copy_advice -> Copy from Advice -> Advice`
   4. `assign_advice_from_instance` 存在的原因是instance列需要电路用户知道其顺序（以便验证者可以在正确的位置**提供公共输入**），并且如果instance column在某个区域中使用，它会干扰`floor Planner` 重新排序区域以提高效率。因此，halo2 设计者将公共输入值从其已知实例列位置复制到区域内的建议单元格中，然后从那里使用它。

7. 在 PLONK（zk-SNARKs）的语境中，`permations`是指一种特定的约束，用于确保向量中元素的顺序得以保留。这在加密应用中特别有用，因为数据的顺序可能会对系统产生重大影响。

   假设一个向量代表交易ID。如果这些ID的顺序发生变化，可能会改变交易的含义，并可能危及系统的完整性。通过强制执行置换约束，PLONK 确保验证者可以验证证明者没有篡改 ID 的顺序。

   PLONK 通过利用一种称为“洗牌证明”的技术来实现这一点。这些证明允许证明者证明他们可以对向量的元素进行洗牌，同时保持它们原来的位置。然后验证者可以检查洗牌向量的有效性，而无需透露实际值或它们的原始顺序。

   置换约束在增强基于 PLONK 的加密系统的安全性和隐私性方面起着至关重要的作用。通过防止数据顺序的操纵，它们有助于保护交易、身份和其他敏感信息的完整性。

   以下是对 PLONK 中置换约束工作原理的简化说明：

   1. **证明者:**
      - 创建一个值向量（例如，交易ID）。
      - 在保留原始和洗牌顺序的情况下对向量进行洗牌。
      - 使用 PLONK 生成证明，该证明演示了洗牌过程，而无需透露实际值或它们的原始顺序。
   2. **验证者:**
      - 从证明者接收洗牌向量和 PLONK 证明。
      - 使用 PLONK 的置换约束验证证明，确保洗牌操作正确执行。
      - 无需了解实际值或它们的原始顺序，可以确认数据完整性得到保持。

   置换约束是 PLONK 安全功能的重要组成部分，能够实现可验证的洗牌，并保护加密应用中数据顺序敏感的特性。



   

   

​    