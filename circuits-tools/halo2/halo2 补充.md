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



   

   

​    