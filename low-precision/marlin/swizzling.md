# Shared Memory Swizzling

## 什么是 Bank Conflict？

### 共享内存的 Bank 结构

GPU 的共享内存（Shared Memory）被划分为 **32 个 bank**，每个 bank 宽度为 4 字节。连续的 4 字节地址被依次分配到不同的 bank：

```
地址 0-3     → Bank 0
地址 4-7     → Bank 1
地址 8-11    → Bank 2
...
地址 124-127 → Bank 31
地址 128-131 → Bank 0  (循环回来)
```

**关键性能指标**：每个 bank 每周期可以提供 **4 字节** 的数据。因此：
- 32 个 bank 并行工作时，峰值带宽 = 32 × 4B = **128 B/cycle**
- 若所有线程只能访问 1 个 bank，带宽 = **4 B/cycle**

当一个 warp（32 个线程）同时访问共享内存时：
- **无冲突**：每个线程访问不同的 bank → 32 个 bank 并行服务 → 128 B/cycle
- **Bank Conflict**：多个线程访问同一 bank 的不同地址 → 该 bank 串行服务 → 带宽骤降

### 矩阵转置：Bank Conflict 的经典案例

考虑一个 32×32 的 float 矩阵存储在共享内存中，我们要对它做转置。

#### 内存布局

矩阵按行优先存储，每个 float 占 4 字节，每行 32 个元素 = 128 字节：

```
                     Bank 0   Bank 1   Bank 2   ...   Bank 31
                       ↓        ↓        ↓              ↓
第 0 行:  ┌─────────┬────────┬────────┬─────────────┬────────┐
          │ [0][0]  │ [0][1] │ [0][2] │     ...     │ [0][31]│
          ├─────────┼────────┼────────┼─────────────┼────────┤
第 1 行:  │ [1][0]  │ [1][1] │ [1][2] │     ...     │ [1][31]│
          ├─────────┼────────┼────────┼─────────────┼────────┤
第 2 行:  │ [2][0]  │ [2][1] │ [2][2] │     ...     │ [2][31]│
          ├─────────┼────────┼────────┼─────────────┼────────┤
   ...    │   ...   │  ...   │  ...   │     ...     │  ...   │
          ├─────────┼────────┼────────┼─────────────┼────────┤
第 31 行: │ [31][0] │[31][1] │[31][2] │     ...     │[31][31]│
          └─────────┴────────┴────────┴─────────────┴────────┘
```

关键观察：**每行正好 128 字节 = 32 个 bank**，所以每一行的第 0 列都在 Bank 0，第 1 列都在 Bank 1，以此类推。

#### 按行读取：无 Bank Conflict ✓

32 个线程同时读取同一行的 32 个元素：

```
Thread 0  → [y][0]  → Bank 0
Thread 1  → [y][1]  → Bank 1
Thread 2  → [y][2]  → Bank 2
...
Thread 31 → [y][31] → Bank 31

→ 32 个 bank 各服务 1 个线程，每个提供 4B，总计 128 B/cycle
```

#### 按列读取：32 路 Bank Conflict ✗

转置时需要按列读取，32 个线程同时读取同一列的 32 个元素：

```
Thread 0  → [0][x]  → Bank x
Thread 1  → [1][x]  → Bank x   ← 同一个 bank!
Thread 2  → [2][x]  → Bank x   ← 同一个 bank!
...
Thread 31 → [31][x] → Bank x   ← 同一个 bank!

→ 只有 Bank x 在工作，每周期服务 1 个线程（4B），带宽仅 4 B/cycle
```

为什么会这样？因为 `[0][x]` 和 `[1][x]` 的地址差了 128 字节（一整行），而 128 字节正好是 32 个 bank 的完整循环，所以它们落在同一个 bank！

### 性能影响

一个 warp 需要读取 32 个 float = 128 字节：

| 访问模式 | Bank Conflict | 实际带宽 | 完成时间 |
|---------|---------------|---------|---------|
| 按行访问 | 无 | 32 × 4B = 128 B/cycle | 1 个周期 |
| 按列访问 | 32 路 | 1 × 4B = 4 B/cycle | 32 个周期 |

按列访问时，32 个线程都挤在同一个 bank，该 bank 每周期只能服务 1 个请求（4 字节），需要 32 个周期才能服务完所有线程。

这意味着朴素的矩阵转置中，按列访问的带宽只有按行访问的 **1/32**！

这就是为什么我们需要 **Swizzling**（数据重排）技术来消除 bank conflict。

---

## Swizzling 如何避免 Bank Conflict

### 方法一：Padding（填充）

最简单的方法是在每行末尾添加一个元素的填充，使每行变成 33 个元素（132 字节）而不是 32 个（128 字节）。

#### 填充后的内存布局

每行 33 个元素 = 132 字节。由于 132 / 4 = 33，而 33 mod 32 = 1，所以每行的起始 bank 会比上一行错开 1 个：

```
第 0 行 (起始于 Bank 0):
  元素: [0][0] [0][1] [0][2] ... [0][31] [PAD]
  Bank:   0      1      2    ...   31      0

第 1 行 (起始于 Bank 1):
  元素: [1][0] [1][1] [1][2] ... [1][31] [PAD]
  Bank:   1      2      3    ...    0      1

第 2 行 (起始于 Bank 2):
  元素: [2][0] [2][1] [2][2] ... [2][31] [PAD]
  Bank:   2      3      4    ...    1      2

第 3 行 (起始于 Bank 3):
  元素: [3][0] [3][1] [3][2] ... [3][31] [PAD]
  Bank:   3      4      5    ...    2      3
```

关键变化：每行的第 0 列所在的 bank 依次递增（0, 1, 2, 3, ...）。

#### 按行读取：无冲突 ✓

32 个线程同时读取第 y 行的 32 个元素：

```
Thread 0  → [y][0]  → Bank (y mod 32)
Thread 1  → [y][1]  → Bank (y+1 mod 32)
Thread 2  → [y][2]  → Bank (y+2 mod 32)
...
Thread 31 → [y][31] → Bank (y+31 mod 32)

→ 32 个线程访问 32 个连续的 bank（循环），全部不同，无冲突！
```

虽然起始 bank 不再是 0，但同一行内的 32 个元素仍然分布在 32 个不同的 bank 中。

#### 按列读取：无冲突 ✓

```
读取第 0 列:
Thread 0  → [0][0] → Bank 0
Thread 1  → [1][0] → Bank 1   ← 错开了 1 个 bank!
Thread 2  → [2][0] → Bank 2   ← 又错开 1 个!
Thread 3  → [3][0] → Bank 3
...
Thread 31 → [31][0] → Bank 31

→ 32 个线程访问 32 个不同的 bank，无冲突！
```

**缺点**：浪费 3% 的共享内存空间（每行多 1 个元素）。

---

### 方法二：XOR Swizzling（异或重排）

更优雅的方法是用 **XOR 运算** 重新映射存储位置，不浪费任何空间。

#### 核心思想

存储元素 `[row][col]` 时，不存在原位置，而是存在 `[row][col XOR row]` 的位置。

```
原始位置          →  Swizzled 位置
[row][col]        →  [row][col XOR row]
```

#### Swizzled 后的内存布局

```
第 0 行: col XOR 0 = col（不变）
  [0][0] [0][1] [0][2] [0][3] ... [0][31]
  Bank 0 Bank 1 Bank 2 Bank 3 ... Bank 31

第 1 行: col XOR 1
  [1][1] [1][0] [1][3] [1][2] ... [1][30]
  Bank 0 Bank 1 Bank 2 Bank 3 ... Bank 31
  (存的是 col=1,0,3,2...30 的数据)

第 2 行: col XOR 2
  [2][2] [2][3] [2][0] [2][1] ... [2][29]
  Bank 0 Bank 1 Bank 2 Bank 3 ... Bank 31

第 3 行: col XOR 3
  [3][3] [3][2] [3][1] [3][0] ... [3][28]
  Bank 0 Bank 1 Bank 2 Bank 3 ... Bank 31
```

#### 按列读取第 0 列：无冲突 ✓

要读取逻辑上的第 0 列，需要从 swizzled 位置读取：

```
逻辑 [0][0] 存在物理位置 [0][0 XOR 0] = [0][0] → Bank 0
逻辑 [1][0] 存在物理位置 [1][0 XOR 1] = [1][1] → Bank 1
逻辑 [2][0] 存在物理位置 [2][0 XOR 2] = [2][2] → Bank 2
逻辑 [3][0] 存在物理位置 [3][0 XOR 3] = [3][3] → Bank 3
...
逻辑 [31][0] 存在物理位置 [31][0 XOR 31] = [31][31] → Bank 31

→ 32 个线程访问 32 个不同的 bank，无冲突！
```

#### 为什么 XOR 有效？

关键性质：对于固定的 `col`，`col XOR row` 对于不同的 `row` 值会产生不同的结果。

```
col = 0 时:
  row=0: 0 XOR 0 = 0  → Bank 0
  row=1: 0 XOR 1 = 1  → Bank 1
  row=2: 0 XOR 2 = 2  → Bank 2
  row=3: 0 XOR 3 = 3  → Bank 3
  ...全部不同！

col = 5 时:
  row=0: 5 XOR 0 = 5  → Bank 5
  row=1: 5 XOR 1 = 4  → Bank 4
  row=2: 5 XOR 2 = 7  → Bank 7
  row=3: 5 XOR 3 = 6  → Bank 6
  ...全部不同！
```

XOR 的双射性质保证了：**无论按行还是按列访问，都不会产生 bank conflict**。

---

### 两种方法对比

| 方法 | 空间开销 | 地址计算 | 行访问 | 列访问 |
|-----|---------|---------|-------|-------|
| Padding | +3% | 简单（+1） | 无冲突 | 无冲突 |
| XOR Swizzling | 0% | 需要 XOR | 无冲突 | 无冲突 |

**XOR Swizzling** 是更优的选择：零空间开销，且 XOR 运算在 GPU 上几乎没有额外延迟。

---

## 矩阵乘法中的 Swizzling

矩阵转置只是一个教学示例，**矩阵乘法（GEMM）** 才是 swizzling 最重要的应用场景。

### Tiled GEMM 的基本流程

计算 C = A × B 时，GPU 将大矩阵分成小块（tile），每个 thread block 负责计算 C 的一个 tile：

```
        K                           N
    ┌───────────┐               ┌───────┐
    │           │               │       │
  M │     A     │       K       │   B   │
    │           │    ┌──────────┤       │
    └───────────┘    │          └───────┘
                     │              ↓
              ┌──────┴──────┐   ┌───────┐
              │  A_tile     │ × │B_tile │ → C_tile
              │  (M×K)      │   │(K×N)  │   (M×N)
              └─────────────┘   └───────┘
```

每次迭代：
1. 从 Global Memory 加载 A_tile 和 B_tile 到 Shared Memory
2. 从 Shared Memory 读取数据做乘加运算
3. 累加到 C_tile

### Bank Conflict 发生在哪里？

问题出在**第 2 步**：从共享内存读取 A_tile 和 B_tile。

假设 A_tile 是 32×32，按行存储在共享内存中：

```
A_tile 在共享内存中的布局（与矩阵转置示例相同）：

         Bank 0   Bank 1   Bank 2   ...   Bank 31
第 0 行:  A[0,0]   A[0,1]   A[0,2]   ...   A[0,31]
第 1 行:  A[1,0]   A[1,1]   A[1,2]   ...   A[1,31]
  ...      ...      ...      ...    ...     ...
第 31 行: A[31,0]  A[31,1]  A[31,2]  ...   A[31,31]
```

#### 计算时的访问模式

计算 C 的一列时，需要用 A 的**一列**去乘 B 的一行：

```
C[i, j] = Σ A[i, k] × B[k, j]

计算 C 的第 j 列（所有 C[0..31, j]）：

  Thread 0  计算 C[0,j]  需要读 A[0, 0..31]   ← A 的第 0 行
  Thread 1  计算 C[1,j]  需要读 A[1, 0..31]   ← A 的第 1 行
  ...
  Thread 31 计算 C[31,j] 需要读 A[31, 0..31]  ← A 的第 31 行
```

看起来每个线程读自己的一行，没问题？

**问题在于具体某一步**：当所有线程同时读取 A 的第 k 列时（用于乘以 B[k, j]）：

```
同时读取 A 的第 k 列：
  Thread 0  → A[0, k]  → Bank k
  Thread 1  → A[1, k]  → Bank k   ← 同一 bank!
  Thread 2  → A[2, k]  → Bank k   ← 同一 bank!
  ...
  Thread 31 → A[31, k] → Bank k   ← 32 路冲突!
```

这与矩阵转置中按列读取的情况**完全相同**！

### Swizzling 解决 GEMM 的 Bank Conflict

对 A_tile 应用 XOR swizzling，存储时将 `A[row][col]` 存到 `A[row][col XOR row]`：

```
读取逻辑上的第 k 列时，实际访问的物理位置：
  Thread 0  → A[0, k XOR 0]  = A[0, k]    → Bank k
  Thread 1  → A[1, k XOR 1]  = A[1, k^1]  → Bank k^1
  Thread 2  → A[2, k XOR 2]  = A[2, k^2]  → Bank k^2
  ...
  Thread 31 → A[31, k XOR 31] = A[31, k^31] → Bank k^31

→ 32 个线程访问 32 个不同的 bank，无冲突！带宽 128 B/cycle
```

### 实际 GEMM 实现中的考量

在真实的高性能 GEMM 实现中（如 cuBLAS、CUTLASS）：

1. **Tile 大小不一定是 32×32**：可能是 64×64、128×32 等，swizzle 模式需要相应调整

2. **向量化加载**：通常使用 `float4`（16 字节）一次加载 4 个 float，此时 bank 的计算方式也要调整

3. **双缓冲（Double Buffering）**：在计算当前 tile 时，预加载下一个 tile，swizzle 同样适用

4. **寄存器分块**：每个线程负责 C 的多个元素，访问模式更复杂，但 swizzle 原理相同

### 小结

| 场景 | 无 Swizzle | 有 Swizzle |
|-----|-----------|-----------|
| 加载 A_tile（按行） | 128 B/cycle | 128 B/cycle |
| 计算时读 A 的列 | 4 B/cycle（32路冲突） | 128 B/cycle |
| 加载 B_tile（按行） | 128 B/cycle | 128 B/cycle |
| 计算时读 B 的行 | 128 B/cycle | 128 B/cycle |

Swizzling 使 GEMM 中 A_tile 的读取带宽提升 **32 倍**，这对计算密集型的矩阵乘法至关重要。

---

## Marlin W4A16 中的 Swizzling

Marlin 是针对 **W4A16**（4-bit 权重 + FP16 激活）量化推理优化的高性能 CUDA kernel。在这个场景中，swizzling 的设计更加复杂。

### W4A16 的数据布局挑战

#### 4-bit 权重的 Packing

4-bit 量化意味着每个权重只占 4 bit，通常将 8 个 4-bit 权重打包成一个 32-bit（int32）：

```
一个 int32 包含 8 个 4-bit 权重：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ w7 │ w6 │ w5 │ w4 │ w3 │ w2 │ w1 │ w0 │
└────┴────┴────┴────┴────┴────┴────┴────┘
  4bit 4bit 4bit 4bit 4bit 4bit 4bit 4bit = 32 bit
```

#### 计算流程

```
1. 从 Global Memory 加载 packed int32 权重到 Shared Memory
2. 从 Shared Memory 读取 int32
3. 解包（unpack）：提取 8 个 4-bit 值，转换为 FP16
4. FP16 权重 × FP16 激活 → 累加
```

### Bank Conflict 的来源

假设权重矩阵 W 的一个 tile 是 128×64（128 行，64 列），每行 64 个 4-bit 权重 = 32 字节 = 8 个 int32。

按行存储时：

```
         Bank 0    Bank 1    Bank 2   ...   Bank 7    Bank 8  ...
第 0 行:  W[0,0:7]  W[0,8:15] W[0,16:23]... W[0,56:63]
第 1 行:  W[1,0:7]  W[1,8:15] W[1,16:23]... W[1,56:63]
  ...
第 127行: W[127,0:7] ...
```

（每个 int32 包含 8 个权重，占 1 个 bank）

**问题**：每行只有 8 个 int32 = 8 个 bank，而一个 warp 有 32 个线程。如果 32 个线程同时访问不同行的同一列位置：

```
Thread 0  → W[0, k]   → Bank (k mod 8)
Thread 1  → W[1, k]   → Bank (k mod 8)   ← 同一 bank!
Thread 2  → W[2, k]   → Bank (k mod 8)   ← 同一 bank!
...
Thread 31 → W[31, k]  → Bank (k mod 8)   ← 同一 bank!

→ 32 个线程访问同一 bank，产生 32 路冲突
```

### Marlin 的 Swizzle 策略

由于每行只有 8 个 bank，简单的 `col XOR row` 无法完全消除冲突。Marlin 采用**分组 XOR**策略：

```
物理地址计算：
  physical_col = logical_col XOR ((logical_row / rows_per_group) & mask)

其中：
  - rows_per_group: 通常是 4 或 8
  - mask: 用于限制 XOR 的范围
```

#### 效果：读取同一逻辑列时

```
读取逻辑列 k（假设 rows_per_group = 4）：
  Thread 0-3   (行 0-3):   physical_col = k XOR 0 = k     → Bank 组 A
  Thread 4-7   (行 4-7):   physical_col = k XOR 1 = k^1   → Bank 组 B
  Thread 8-11  (行 8-11):  physical_col = k XOR 2 = k^2   → Bank 组 C
  Thread 12-15 (行 12-15): physical_col = k XOR 3 = k^3   → Bank 组 D
  ...

→ 不同行组访问不同的 bank 组，冲突从 32 路降低到 4 路
```

### 为什么不能完全消除冲突？

在 W4A16 场景中，完全消除 bank conflict 更困难，核心原因是**数据密度高**：4-bit packing 使得每行只有 8 个 bank（64 列 ÷ 8 权重/int32 = 8 个 int32），远少于标准 GEMM 的 32 个 bank。

Marlin 的设计目标是在有限的 bank 数量下，尽可能分散冲突。

### Marlin Swizzle 的实际效果

| 配置 | 无 Swizzle | Marlin Swizzle |
|-----|-----------|----------------|
| Bank Conflict | 32 路 | 4 路 |
| 读取带宽利用率 | ~3% | ~25% |
| 整体 Kernel 性能 | 基准 | ~4-6x 提升 |

虽然没有完全消除冲突，但将 32 路冲突降到 4 路，配合其他优化（异步加载、软件流水线等），Marlin 在 W4A16 推理中实现了接近理论峰值的性能。

### 小结

| 对比项 | Standard GEMM | Marlin W4A16 |
|-------|---------------|--------------|
| 数据类型 | FP16/FP32 | 4-bit packed + FP16 |
| 每行 bank 数 | 32（128B / 4B） | 8（32B / 4B） |
| Swizzle 方案 | 简单 XOR | 分组 XOR |
| 最终冲突 | 0 路 | 4 路（可接受） |

Marlin 展示了在 4-bit 高密度 packing 约束下，如何通过分组 swizzle 最大化共享内存带宽利用率。
