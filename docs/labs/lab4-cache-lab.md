# Lab4: Cache Lab

**DDL：2026 年 12 月 11 日 23:59**

## 实验概述

本实验配套 CS:APP 第 6 章，帮助大家理解高速缓存对程序性能的影响。**四个部分均为必做**。

- **Part A**：用 C 写一个缓存模拟器 `csim.c`，读入 valgrind 生成的访存 trace，输出命中、缺失和替换次数。
- **Part B**：优化矩阵转置 `trans.c`，让模拟缓存上的 miss 尽量少。
- **Part C**：优化矩阵乘法 `mul.c`，让模拟缓存上的 miss 尽量少。
- **Bonus**：给定一个参数未知的缓存，只能通过读写它来推断它的全部参数，写在 `solver.c` 中。

| 部分 | 自动评测分数 |
|---|---:|
| Part A | 35 |
| Part B | 40 |
| Part C | 25 |
| Bonus | 10 |
| 合计 | 110 |

实验报告单独计分。

## 实验环境

请在课程服务器上完成并测试。需要 `gcc`、`make`、`python3` 和 `valgrind`：

```bash
gcc --version && make --version && python3 --version && valgrind --version

# 如果未安装：
sudo apt-get update
sudo apt-get install gcc
sudo apt-get install make libc6 libc6-dev libc6-dev-i386
sudo apt-get install python3
sudo apt-get install valgrind
```

每次修改代码后重新编译：

```bash
make clean && make
```

## Part A：缓存模拟器

??? note "可选阅读 · 一份访存记录，多种缓存配置"

    1970 年，Mattson 等人在《IBM Systems Journal》介绍了一种评估存储系统的方法。对一类存储层次结构，只需扫描一遍地址序列，就能算出多种配置下的缺失率。对于采用 LRU 的全相联缓存，可以按最近访问的先后给内存块排队。一个块再次被访问时，它在队列里的位置就说明了缓存至少要容纳多少块，才能把它保留到这一次访问。

    本节也把记录访存和模拟缓存分成了两步。Valgrind 先记录程序访问过的地址，`csim` 再按指定的配置逐条重放。固定同一份 trace，就可以观察块大小或相联度改变后，哪些访问仍然命中，哪些变成了缺失。 [Mattson 等人的论文](https://doi.org/10.1147/sj.92.0078)

在 `csim.c` 中实现一个缓存模拟器，用法与参考程序 `csim-ref` 相同：

```
./csim-ref [-hv] -s <s> -E <E> -b <b> -t <tracefile>
```

- `-s`：组索引位数，组数 S = 2^s
- `-E`：相联度，每组的行数
- `-b`：块偏移位数，块大小 B = 2^b
- `-t`：trace 文件
- `-h`、`-v`：可选，帮助信息与逐条输出

`csim-ref` 会输出 cache 的命中、缺失、替换次数，比如：

```bash
$ ./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace
hits:4 misses:5 evictions:3

$ ./csim-ref -v -s 4 -E 1 -b 4 -t traces/yi.trace
L 10,1 miss
M 20,1 miss hit
L 22,1 hit
S 18,1 hit
L 110,1 miss eviction
L 210,1 miss eviction
M 12,1 miss eviction hit
hits:4 misses:5 evictions:3
```

`traces/` 下的文件由 valgrind 的 lackey 工具生成，每行一次访存：

```
I 0400d7d4,8
 M 0421c7f0,4
 L 04f6b868,8
 S 7ff0005c8,8
```

格式为 `[空格]操作 地址,大小`。`I` 是取指令，`L` 是读数据，`S` 是写数据，`M` 是修改数据（一次读加一次写）。`I` 行开头没有空格，其他行开头有一个空格。地址是 64 位十六进制数，大小以字节为单位。

**你的所有实现都在 `csim.c` / `csim.h` 中。**你的全局变量和函数需要定义在 `csim.h` 中，你的函数实现需要在 `csim.c` 中。

要求：

1. 编译不能有 warning（Makefile 使用 `-Wall -Werror`）。
2. 对任意 s、E、b 都要正确，用 `malloc` 按参数分配缓存结构，结束前释放。
3. 替换策略为 LRU。
4. 忽略 `I`。`M` 等于对同一地址先读后写。可以假设每次访存都不跨块，因此不用考虑大小字段。
5. 在 `main` 结束前调用 `printSummary(hits, misses, evictions)` 输出结果。
6. 只能用 C 实现，不能调用 `csim-ref`，不能针对测试数据硬编码，助教会检查源码。

评分：

```bash
./test-csim
```

共 8 个测例，前 7 个原始分各 3 分，最后一个原始分 6 分。每个测例中 hits、misses、evictions 各占三分之一。`test-csim` 给出满分 27 的原始分，`driver.py` 将原始分乘以 `35/27`，折算为 Part A 的得分。

**hints**

- 可以使用 `malloc` 和 `free` 构造 cache。
- 你可以使用 `csim-ref` 来检查你的实现是否正确，通过开启 `verbose` 模式可以更好地 debug。
- LRU 算法可以简单地使用计数器的实现方式。

## Part B：矩阵转置

??? note "可选阅读 · 矩阵的一行多留一个位置"

    NVIDIA 曾用矩阵转置演示 GPU 上的内存冲突。在这个例子里，GPU 的共享内存分为 32 个分区（bank），相邻元素依次分布到不同的 bank。数据块每行恰好有 32 个元素时，同一列的元素相隔 32 个位置，刚好又落回同一个 bank。32 个线程同时读一列时，这些访问需要排队。

    在每行末尾多留一个位置后，列内元素的间隔从 32 变成 33，下一行的同列元素就会落到下一个 bank，原来的冲突也随之消失。这一列额外空间只改变了数据在共享内存里的布局。GPU 的 bank 和实验里的 CPU 缓存组是两种机制，但这两个案例可以引出同一个结论：矩阵宽度会改变地址落到哪里。[NVIDIA 的矩阵转置案例](https://developer.nvidia.com/blog/efficient-matrix-transpose-cuda-cc/)

在 `trans.c` 中实现：

```c
char transpose_submit_desc[] = "Transpose submission";
void transpose_submit(int M, int N, int A[N][M], int B[M][N]);
```

把 N 行 M 列的矩阵 A 转置后存入 B，使下面这个缓存上的 miss 尽量少：s = 5、E = 1、b = 5，即 32 组、直接映射、32 字节块，共 1KB。

你可以在 `trans.c` 里写多个转置函数，并在 `registerFunctions()` 中用 `registerTransFunction` 注册，评测程序会逐个测试。评分只看描述字符串为 `"Transpose submission"` 的那个函数，不要修改这个字符串。

要求：

1. 你的代码在编译时不能存在warning。
2. 在每个矩阵转置函数中，你至多能定义12个int类型的局部变量（不包括循环变量，但你不能将循环变量用作其他用途），且不能使用任何全局变量。你不能定义除int以外类型的变量。你不能使用malloc等方式申请内存块。你可以使用int数组，但等同于数组大小的数量的int类型变量也同样被计入。
3. 你不能使用递归。
4. 你只允许使用一个函数完成矩阵转置的功能，而不能在函数中调用任何辅助函数。
5. 你不能修改原始的矩阵A，但是你可以任意修改矩阵B。
6. 你可以定义宏。

为什么限制局部变量：局部变量位于寄存器或栈上，评测不统计栈上的访存。没有这条限制，就可以把数据先放进局部变量来躲开缓存。

评分：

我们只测三种尺寸，因此你可以针对它们分别优化：

| 尺寸 | 分值 | 满分 | 零分 |
|---|---:|---|---|
| 32 × 32（`-M 32 -N 32`） | 12 | miss ≤ 300 | miss ≥ 600 |
| 64 × 64（`-M 64 -N 64`） | 12 | miss ≤ 1300 | miss ≥ 2000 |
| 61 × 67（`-M 61 -N 67`） | 16 | miss ≤ 2000 | miss ≥ 3000 |

两条线之间按 miss 数线性给分，每个尺寸的得分保留一位小数。函数结果不正确时该尺寸为 0 分。

```bash
./test-trans -M 32 -N 32
```

`test-trans` 会先运行 `tracegen` 检查正确性，再用 valgrind 记录访存并交给 `csim-ref` 计数。第 i 个注册函数的 trace 保存在 `trace.fi`，可以逐条查看命中情况：

```bash
./csim-ref -v -s 5 -E 1 -b 5 -t trace.f0 > result.txt
```

**hints**

- 分块（blocking）是这部分的核心技巧，可以参考 [Using Blocking to Increase Temporal Locality](http://csapp.cs.cmu.edu/public/waside/waside-blocking.pdf)。
- 直接映射缓存上冲突 miss 很多，尤其是沿对角线的元素。先算清楚 A、B 的元素分别落在哪一组，再设计访问顺序。
- 先在纸上估算 miss 数，再用上面提到的 `-v` 命令的输出核对。

## Part C：矩阵乘法

??? note "可选阅读 · 让搬进缓存的数据多参与几次计算"

    矩阵转置只需把每个输入元素放到对应的输出位置。矩阵乘法里，同一个输入元素会参与多个输出元素的计算。例如，固定 `i` 和 `k` 后，`A[i][k]` 与 `B` 的第 `k` 行中不同列的元素相乘，乘积分别累加到 `C` 第 `i` 行的不同位置。如果这些计算集中进行，刚取来的 `A[i][k]` 就能接着用。

    高性能矩阵乘法库 GotoBLAS 把这个思路用到了整块数据上。它按存储层次组织计算，还会把部分矩阵块打包到连续的内存区域，方便后续反复读取。打包增加了一次复制，复用次数足够多时，这笔开销就可以分摊到多次乘加上。矩阵乘法适合练习缓存优化，正是因为计算同一个乘积时，运算顺序会影响数据被反复搬运的次数。 [Goto 与 van de Geijn 的论文](https://doi.org/10.1145/1356052.1356053)

在 `mul.c` 中实现：

```c
char mul_submit_desc[] = "Multiply submission";
void mul_submit(int M, int N, int A[N][M], int B[M][N], int C[N][N]);
```

计算 C = A × B，其中 A 为 N × M，B 为 M × N，C 为 N × N。缓存参数与 Part B 相同。

最朴素的写法如下：

```c
for (i = 0; i < N; i++)
    for (j = 0; j < N; j++) {
        tmp = 0;
        for (k = 0; k < M; k++)
            tmp += A[i][k] * B[k][j];
        C[i][j] = tmp;
    }
```

最内层循环沿行读 A，却沿列读 B：每读一个 B 元素都要跳过一整行，取进来的缓存块只用到一个元素就可能被替换。调换循环顺序能减少一部分 miss，但三个矩阵加起来远大于 1KB，仍然会频繁冲突。和 Part B 一样，可以把矩阵分成能放进缓存的小块，按块计算：

$$
C_{ij} = \sum_k A_{ik} B_{kj}
$$

只做到这一步还不够，需要继续分析冲突 miss。

要求：

1. 编译不能有 warning。
2. 最多定义 16 个 `int` 类型的局部变量。循环变量不计入，但不能把循环变量挪作他用。`int` 数组按元素个数计入。不能定义 `int` 以外类型的变量。
3. 不能使用全局变量、`malloc` 和递归。
4. 只能用一个函数完成乘法，不能调用辅助函数。可以定义宏。
5. 不能修改 A 和 B（评测程序会检查）。C 可以随意修改，但 C 的初始内容是随机值，每个元素都必须写入正确结果。

评分：

只测 32 × 32：

```bash
./test-mul -M 32 -N 32
```

结果不正确时得 0 分。结果正确时，miss ≤ 3999 得 25 分，miss ≥ 35000 得 0 分。两条线之间按 `25 × (35000 − miss) / (35000 − 3999)` 线性给分。例如，20000 次 miss 约得 12.096 分。

第 i 个注册函数的 trace 保存在 `trace.mi`。

## Bonus：缓存逆向

??? note "可选阅读 · 从访问时间读出缓存规则"

    真实 CPU 会公开一些缓存参数，但具体的替换规则未必写在手册里。2020 年，CacheQuery 的研究者在真实处理器上测量访存耗时，据此区分命中和缺失。他们比较不同访问序列得到的反馈，再用这些观测推断确定性的替换规则，发现了两种此前没有公开记录的策略。

    两种策略可以在一串访问里表现得完全一样。某次替换选中了不同的缓存行以后，接下来访问这些数据时，差异才可能显现。本题的接口直接返回命中或缺失，省去了从计时数据判断命中的步骤。 [CacheQuery 论文](https://arxiv.org/abs/1912.09770)

Part A 是已知缓存参数，求每次访问是否命中。这道题反过来：给你一个参数未知的缓存，你只能访问它、看结果是命中还是缺失，要推断出它的全部参数。

你需要在 `solver.c` 中实现 `solve()`。它可以调用以下函数（见 `solver.h`）：

```c
int  cache_load(uint32_t addr);    /* 读 addr，命中返回 1，缺失返回 0 */
int  cache_store(uint32_t addr);   /* 写 addr，命中返回 1，缺失返回 0 */
void cache_flush(void);            /* 所有行置为无效，替换状态复位 */
void report(int b, int s, int E, int policy, int write_allocate);
```

`solve()` 每次面对一个新的神秘缓存。探测完后调用一次 `report()` 给出答案。探测次数不计分，每个缓存最多 1,000,000 次（`cache_load` 和 `cache_store` 都算，`cache_flush` 不算），限时 30 秒。

**神秘缓存：**

地址 32 位。缓存有 `2^s` 组，每组 `E` 行，块大小 `2^b` 字节。低 `b` 位是块内偏移，接下来 `s` 位是组索引，其余是标记。你要求出：

| 参数 | 取值 |
|---|---|
| `b` | 3～7 |
| `s` | 1～9 |
| `E` | 1, 2, 3, 4, 5, 6, 8, 12, 16 |
| `policy` | 下面三种替换策略之一（E = 1 时不评分，填什么都可以） |
| `write_allocate` | 1：写缺失时调入该块；0：写缺失时缓存不变 |

**访问规则：**

- 命中时，更新这一行的替换状态。读和写都一样。
- 读缺失，或者 `write_allocate = 1` 时的写缺失：如果这一组有无效行，使用编号最小的无效行；否则按替换策略选一行换出。新块调入后同样更新替换状态。
- `write_allocate = 0` 时的写缺失：缓存和替换状态都不变。
- `cache_flush()`：所有行置为无效，替换状态恢复初始值（没有任何访问历史）。RANDOM 的随机数发生器不复位。

**替换策略：** 行号为 0～E−1。"访问"指一次命中或一次调入。

| 名字 | 定义 |
|---|---|
| `POLICY_LRU` | 换出最久没有被访问的行。 |
| `POLICY_FIFO` | 换出最早调入的行。命中不影响顺序。 |
| `POLICY_RANDOM` | 在 E 行中均匀随机选一行。 |

要求：

1. 编译不能有 warning。
2. `solver.c` 只能调用上面的缓存函数，以及内存与字符串函数（`memset`、`memcpy`、`memmove`、`memcmp`、`strlen`、`strcmp`）、输出函数（`printf`、`puts`、`putchar`、`fprintf` 到 `stdout`/`stderr` 等）、`rand`、`srand`、`abs`、`qsort`、`exit`、`assert`。不能打开文件、读环境变量、创建进程，不能用内联汇编，不能定义 `main()`。
3. 全局变量和 `static` 变量可以用：每个缓存都在一个新进程里求解。
4. 每个缓存只能调用一次 `report()`。
5. 不能针对评分数据硬编码。正式评分的缓存由你的学号和一个只有助教持有的密钥生成，你在本地无法得到它们。
6. 缓存逆向部分的实现写在 `solver.c` 中。

`test-solver` 运行前会用 `nm` 检查 `solver.o`，违反第 2 条时整题为 0 分。

评分：

每位同学有 8 个缓存，由学号生成，组成对每个人都相同：

| 缓存 | 替换策略 |
|---|---|
| 0, 1 | LRU |
| 2, 3 | FIFO |
| 4, 5 | RANDOM |
| 6, 7 | 直接映射（E = 1） |

每一对中一个是写分配、一个不是。b、s、E 在上面的范围内随机。`solve()` 不知道自己面对的是第几个缓存。

| 项目 | 分值 | 条件 |
|---|---:|---|
| 几何参数 | 4 | b、s、E 全对的缓存数 / 8 |
| 写分配 | 1 | 几何参数对且 `write_allocate` 对的缓存数 / 8 |
| 替换策略 | 5 | 几何参数对且 `policy` 对的缓存数 / 6（E = 1 的两个不算） |

共 10 分。超时、超过探测上限、没有调用 `report()` 的缓存，该缓存三项都不得分。

测试：

```bash
make clean && make
./test-solver -i 你的学号            # 你的学号的 8 个练习缓存
./test-solver -n 2 -v               # 只跑第 2 个，并打印每一次探测
./test-solver -c 5,4,8,FIFO,0       # 指定一个缓存：b,s,E,policy,write_allocate
python3 driver.py -B                # 只跑 Bonus，按评分规则估计分数（可加 -i 你的学号）
```

`test-solver` 会列出每个缓存的真实参数和你的答案。8 个缓存时，`solve()` 打印的内容保存在 `solver.out.0`～`solver.out.7`。

练习缓存与正式缓存来自同一个分布，只是密钥不同，所以与正式缓存不同。一个在各种参数上都正确的求解器，在正式评分中也会得到同样的分数。建议用几个不同的学号和 `-c` 多测一些情况，特别是范围的边界。

**hints**

- **块偏移位数 `b`**。`solver.c` 已给出示例。观察偏移量第一次跨出已装入的块时，读取结果怎样变化。
- **相联度 `E` 和组索引位数 `s`**。清空缓存，按固定地址间隔读入一批不同块，再按原顺序读一遍。第二轮全部命中，说明第一轮结束时这些块都还在。先想怎样构造一批落在同一组的块来测每组容量，再调节间隔，观察这些块何时开始发生冲突。
- **写分配**。清空后只需两次探测。想想哪一次访问能揭示写缺失是否调入了目标块。
- **替换策略**。填满一组后，一次命中会改变下次应换出的块吗？试着设计访问序列，让 LRU 和 FIFO 在某次回读上给出不同结果。重复探测可以降低 RANDOM 偶然表现得像确定性策略的概率。也可以参考 Part A 模拟 LRU 和 FIFO，再把预测与实际探测结果比较。

报告中的 Bonus 可以进一步说明以下内容。

- 你如何求出每一个参数，以及为什么在所有允许的取值下都正确；
- 给出一串访问，说明 E = 4 时 LRU 与 FIFO 的结果在哪一步开始不同。

## 实验报告

报告需要包含以下内容：

- Part A、B、C 和 Bonus 的实现和相关分析
- 你的代码运行效果展示

在确保包含必需的内容的前提下，有任何你想写的内容都可以。

## 总评测

```bash
make clean && make
python3 driver.py
```

`driver.py` 依次运行 `test-csim`、`test-trans`（三种尺寸）、`test-mul` 和 `test-solver`。`Total points` 汇总 Part A、B、C 和 Bonus 的自动评测分数，满分 110。总分保留三位小数。加 `-B` 只跑 Bonus（不需要 valgrind）。

## 提交

1. 打开课程发放的 CacheLab 学生模板，在 GitHub 点击 **Use this template → Create a new repository**，在自己的账号下创建个人实验仓库，再克隆个人仓库。
2. 完成实验后，确保你的仓库根目录有四份源码 `csim.c`、`trans.c`、`mul.c`、`solver.c` 和一份 PDF 实验报告。请在截止前将这些文件提交并推送到默认分支。
3. 在截止前，到 eLearning 课程的「作业」页面提交自己的 CacheLab 仓库链接。打开该链接，确认 GitHub 默认分支上能看到最新代码，且报告可以打开。
4. 从截止时间起，个人实验仓库应至少保持 **Public** 24 小时，供助教收取。

## 参考资料

- CMU 原版实验：http://csapp.cs.cmu.edu/3e/labs.html
- 本实验参考 2025 年的 Cachelab 实验
