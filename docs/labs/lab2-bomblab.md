---
title: "Lab2：BombLab"
---

# Lab2: BombLab

**DDL：2026 年 10 月 31 日**（具体截止时刻以 E-Learning 作业页面为准）

## 实验简介

本实验是 CSAPP 第三章配套实验。你需要用反汇编和 GDB 分析一个没有提供源码的程序，理解其中的函数调用、分支、循环、递归和指针操作，再用 C 语言写出对应的解法。

每位同学领取自己的二进制炸弹 `bomb`，完成六个正式关卡，并提交 `defuser.c` 和实验报告。正式六关考点相同，但不同同学的常量、运算和数据布局可能不同。另有一个选做的 Honor。

终端中的故事是《这里还没有明天》：你和林夏受邀进入一座每天恢复旧存档的 AI 城市，逐项完成设备交接。六关结束就有完整结局，Honor 提供一份额外的旧档案。剧情不影响题目或评分；运行 `./bomb` 可看到彩色展示，自动测试和输出重定向只保留测试信息。设置 `NO_COLOR=1` 可以关闭颜色。

> **本实验提交的是解法程序，不是六个固定密码。** 每关会给出一个数值 challenge，正确口令随它变化。你需要在 `defuser.c` 中实现从 challenge 算出口令的函数；助教会用未公开的输入检查解法。

基础分为 110 分，Honor 最多额外加 10 分，最高总分为 120 分。具体截止时刻、迟交规则和报告检查安排以 E-Learning 的课程公告及本实验作业页面为准。

!!! tip "前排提示"

    本实验的工作量可能较大，建议尽早开始，不要把调试和报告留到截止前。可以在课程允许的范围内合理使用 AI 工具辅助理解汇编、排查错误和整理思路，但要自己验证结果，能解释提交的代码与分析过程，并在报告中说明使用情况。

## 部署实验环境

### 领取作业并克隆仓库

使用 [BombLab 学生模板](https://github.com/ICS-26Fall-FDU/BombLab) 创建自己的仓库，放在自己的 GitHub 账号下，名称使用 `ics-2026-bomblab`。助教会根据你在 E-Learning 提交的仓库链接收取作业，不需要预先登记 GitHub 账号。

获取仓库后克隆到本地：

```sh
git clone <仓库地址>
cd ics-2026-bomblab
```

将 `<仓库地址>` 换成自己的仓库链接，不要直接在模板仓库中提交作业。

### 环境要求

在 Linux x86-64 环境中完成实验，例如课程服务器、x86-64 Ubuntu 虚拟机或 x86-64 WSL。需要 GCC、GNU Make、Python 3.10+、GDB 和 binutils（包含 `objdump`）。

Ubuntu / Debian 环境可以安装：

```sh
sudo apt-get update
sudo apt-get install gcc make python3 gdb binutils
```

在服务器上以 root 用户登录时（`whoami` 输出 `root`，命令提示符以 `#` 结尾），不需要 `sudo`，直接执行：

```sh
apt-get update
apt-get install -y gcc make python3 gdb binutils
```

本实验的炸弹是 64 位 Linux 程序，不能直接在 Windows、macOS 或 ARM Linux 上运行。可以用 `uname -sm` 检查环境；本实验需要的输出是 `Linux x86_64`。

### 领取个人炸弹并编译

在 `config.txt` 中填写自己的学号，例如：

```text
student_id = 243xxxx0175
```

这里只是示例，请替换成自己的学号。学号为 11 位数字或英文字母；含字母的学号直接填写，保留大小写和前导零，不要转成数字。

在仓库根目录执行：

```sh
make bomb
make
```

`make bomb` 下载个人炸弹，`make` 编译解法程序 `defuser`。两条命令没有报错，且根目录出现 `bomb` 和 `defuser`，就可以开始实验。下载失败的处理方法见下方[常见问题](#make-bomb)。

领取成功后，修改 `defuser.c` 只需要重新运行 `make`，不用再次领取炸弹。已有 `bomb` 时，`make bomb` 也不会重复下载。

为使 GitHub Actions 能运行公开测试，首次领取后把个人炸弹加入版本管理：

```sh
git add -f bomb
```

`config.txt` 只用于领取。已有炸弹的编译、运行和测试不再需要学号配置，你可以清空学号或删除配置，不要求提交；`package.json` 也不需要提交。若不希望把配置中的学号提交到 GitHub，请在首次提交前处理，之后删除文件不会清除已有 Git 历史。

请使用自己的炸弹。最终评分脚本会根据 E-Learning 提交者的学号，使用助教保存的对应 `bomb` 测试解法，不根据仓库中的配置或上传的二进制决定题目。

## 主要文件

以下路径均相对于学生仓库根目录：

| 文件或目录 | 用途 |
| --- | --- |
| `bomb` | 下载的个人炸弹，正式关卡不提供源码 |
| `defuser.c` | <span class="text-red">唯一需要编写的解法代码文件</span> |
| `defuser.h` | 求解函数的接口声明 |
| `input.txt`（自行生成，可选） | 调试时保存口令，供 GDB 用 `< input.txt` 重复读取；不随模板提供，不需要提交 |
| `config.txt` | <span class="text-red">首次领取时填写学号</span>，领取后可清空或删除 |
| `tutor/` | [tutor 教学](https://github.com/ICS-26Fall-FDU/BombLab/blob/main/tutor/README.md)，包含源码和参考解法 |
| `teaching/` | [配套 C 小例子](https://github.com/ICS-26Fall-FDU/BombLab/blob/main/teaching/README.md)，用于练习六关涉及的知识点 |
| `defuser_main.c`、`challenge.c`、`cli.c` 等 | 公共驱动程序，不需要修改 |
| `public_test.py`、`tests/public.json` | 公开测试程序和 seed，包括正式六关和选做 Honor |
| `.github/workflows/public-tests.yml` | GitHub Actions 公开自测工作流 |

## 实验内容

### 正式六关

每关检查一行口令。正确时输出 `Phase N defused.`；错误时输出 `BOOM!!!` 并退出。爆炸不会扣分，也不会被记录，可以反复调试。

| 关卡 | 主要知识点 |
| --- | --- |
| 第 1 关 | 函数调用、参数传递、返回值 |
| 第 2 关 | 数组、循环、递推关系 |
| 第 3 关 | switch 分支与跳转表 |
| 第 4 关 | 递归调用与返回过程 |
| 第 5 关 | 结构体布局、指针、链表遍历 |
| 第 6 关 | 函数指针表与间接调用 |

六关可以分别运行和调试，不必按顺序完成。暂时卡住时，可以先分析另一关。

### 编写解法

只修改 `defuser.c` 中的解法，可以在这个文件中添加辅助函数。不要修改接口、驱动程序、测试程序、工作流或个人炸弹。评分时只取你的 `defuser.c`，与助教保存的驱动程序一起编译。

例如，第 1 关的接口是：

```c
int solve_phase_1(uint64_t challenge, char *out, size_t cap);
```

根据 challenge 算出口令，写入 `out`；字符串连同结尾的 `\0` 不能超过 `cap` 字节。成功时返回 0，尚未实现或发生错误时返回非零。

正式六关的口令是一行用空格分隔的无符号十进制整数。`out` 中不要加换行，驱动程序会自动添加；调试信息写到 stderr，写到 stdout 会混入口令。

模板中的 `defuser.c` 已经写好每关的框架：把算出的整数依次存入 `values`，把 `count` 改为整数个数，`write_answer` 会按上述格式写入 `out` 并检查截断。需要填写的位置都标有 `// TODO`；`count` 为 0 时视为尚未实现。

???+ example "示例：从汇编到求解函数"

    下面以 tutor 教学为例（不计分，`tutor/` 中有完整源码），演示从汇编到求解函数的过程。在 `phase_00` 中，challenge 先被保存到 `rbx`，读入两个整数后有这样几行：

    ```text
    shr    $0x5,%rbx            # challenge 右移 5 位
    movzbl %bl,%edi             # 只保留低 8 位，相当于 & 255
    xor    $0x2a,%edi           # 异或 42，得到第一个数
    cmp    %edi,-0x20(%rbp)     # 与输入的第一个整数比较
    ...
    call   tutorial_mix         # 以第一个数为参数
    cmp    %eax,-0x1c(%rbp)     # 返回值与输入的第二个整数比较
    ```

    `tutorial_mix` 只有一条 `lea 0x7(%rdi,%rdi,2),%eax`，即 `x + 2x + 7`。所以口令的两个数是：

    - 第一个数：`((challenge >> 5) & 255) ^ 42`
    - 第二个数：`3 * 第一个数 + 7`

    写成求解函数，也就是 `tutor/phase0_solution.c`：

    ```c
    #include <inttypes.h>
    #include <stdio.h>

    int solve_phase_0(uint64_t challenge, char *out, size_t cap) {
        uint32_t first = (uint32_t)((challenge >> 5) & UINT64_C(255)) ^ UINT32_C(42);
        uint32_t second = first * UINT32_C(3) + UINT32_C(7);
        int size = snprintf(out, cap, "%" PRIu32 " %" PRIu32, first, second);
        return size < 0 || (size_t)size >= cap;
    }
    ```

    最后一行检查 `snprintf` 是否出错或被截断；正式关卡中这一步由模板里的 `write_answer` 完成。正式关卡也一样：要恢复出对任意 challenge 都成立的计算关系，而不是记下某一次运行时比较的值。

注意汇编中的运算宽度。32 位无符号加、减、乘会按模 2³² 回绕；不要将它们恢复成无限精度计算，也不要依赖 C 的有符号溢出。

解法只应完成计算。可以使用 `snprintf` 等向内存缓冲区写入结果的函数，但不要读写文件、访问网络、创建进程、调用外部程序或使用内联汇编。正式评分会检查禁用调用，并在隔离环境中编译和运行。

## 实验引导

### GDB 教学

GDB 是 GNU 调试器。本实验用它让炸弹停在某条指令之前，查看此刻的寄存器和内存，验证从汇编中读出的推测。下面介绍两种分析方法，以及常用的 GDB 命令。

!!! warning "调试提醒：不要修改程序数据"

    请不要使用 `set` 修改炸弹中的变量、寄存器或内存。调试时应观察原程序的运行过程，通过改程序数据得到的“通关”不算有效结果。

    下面示例中的 `set $challenge = $rsi` 只是把观察到的值保存在 GDB 的便利变量中；`set pagination off` 也仅改变调试器的显示设置。这两类命令可以正常使用。

GDB 命令大多可以缩写，下面各表的“简写”一列给出了常用写法；直接按回车会重复上一条命令，连续单步时很方便。忘记用法时，在 GDB 中执行 `help 命令名`。

#### 两种调试方式

分析一关时，通常交替使用两种方式：

| 方式 | 做法 | 适合回答的问题 |
| --- | --- | --- |
| 方法 1：离线分析 | 用 `objdump` 或 `gdb -batch` 把汇编写进文件，在编辑器中阅读、搜索、标注 | 调用了哪些函数，分支、循环和比较在哪里 |
| 方法 2：GDB 在线调试 | 在 GDB 中运行程序，停在某条指令之前，查看寄存器和内存 | 某一时刻寄存器和内存里的具体值是多少 |

导出的汇编只有指令，看不到运行时的值；在线调试能看到值，但一次只能看一个时刻。下面用公开的示例关卡 `phase_00` 演示两者如何配合，先在 `tutor/` 目录中执行 `make`。

#### 方法 1：离线分析 { .bomblab-method }

导出汇编，推测计算关系：

```sh
objdump -d --no-show-raw-insn --disassemble=phase_00 ./bomb > phase_00.S
```

在 `phase_00.S` 中可以找到“编写解法”示例里的那段指令：challenge 经过右移、取低 8 位和异或后放进 `edi`，再由 `cmp %edi,-0x20(%rbp)` 与输入的第一个整数比较。由此推测第一个数是 `((challenge >> 5) & 255) ^ 42`。记下这条比较指令的地址，例如 `4014d6`；地址以自己导出的汇编为准。

#### 方法 2：GDB 在线调试 { .bomblab-method }

用占位输入运行，停在比较指令处查看两边的值（省略了部分输出）：

```text
$ gdb ./bomb
(gdb) break *phase_00
(gdb) break *0x4014d6
(gdb) run --seed 42 --phase 0 < phase0-input.txt
Breakpoint 1, 0x0000000000401460 in phase_00 ()
(gdb) set $challenge = $rsi
(gdb) p/x $challenge
$1 = 0xbdd732262feb6e95
(gdb) continue
Breakpoint 2, 0x00000000004014d6 in phase_00 ()
(gdb) p/u $edi
$2 = 94
(gdb) x/1uw $rbp-0x20
0x7fffffffd3f0:	0
(gdb) p/u (($challenge >> 5) & 255) ^ 42
$3 = 94
```

- 在入口处用 `set $challenge = $rsi` 把 challenge 存进 GDB 的便利变量。`rsi` 之后会被改写，停在比较指令时它已经不是 challenge 了，所以要在入口先记下。
- 停在 `cmp` 时，`edi` 中是程序算出的期望值 94，`$rbp-0x20` 处是输入的第一个整数，也就是占位值 0。
- 在 GDB 中按推测的公式计算，同样得到 94，说明从汇编读出的计算关系是对的。占位输入与期望值不相等，这次运行会失败，但需要的信息已经拿到了。

换一个 seed 再做一次，结果仍然一致，就可以把公式写进求解函数。正式关卡也按这个思路：先从导出的汇编中找出关键的计算和比较，再在线停在那里核对。命令的详细用法见下面各小节。

#### 启动与退出

```sh
gdb --args ./bomb --seed 42 --phase 1
```

`--args` 把后面的内容都当作被调试程序的参数。此时程序还没有开始运行，先设好断点，再用 `run` 启动。

下面的 `input.txt` 指存有口令的文本文件；`< input.txt` 让程序从文件读取，而不必每次手动输入。只调试一关时，文件里只需一行口令。

| 命令 | 简写 | 作用 |
| --- | --- | --- |
| `run` | `r` | 开始运行，遇到断点时停下；程序等待输入时，直接在 GDB 所在的终端里输入口令 |
| `run --seed 42 --phase 1 < input.txt` | `r --seed 42 --phase 1 < input.txt` | 重新指定参数，并从文件读取口令；带重定向时要把参数写完整 |
| `kill` | `k` | 结束这一次运行，断点保留，可以再次 `run` |
| `quit` | `q` | 退出 GDB |
| `set pagination off` | `set pag off` | 关闭分页，输出较长时不再停下等待回车 |

在部分容器中启动时会提示 `Error disabling address space randomization: Operation not permitted`，不影响本实验，可以忽略。

#### 断点

| 命令 | 简写 | 作用 |
| --- | --- | --- |
| `break *phase_01` | `b *phase_01` | 在函数的第一条指令停下 |
| `break *0x401234` | `b *0x401234` | 在指定地址停下，地址从自己炸弹的反汇编中读取 |
| `tbreak *0x401234` | `tb *0x401234` | 只生效一次的断点 |
| `info breakpoints` | `i b` | 列出所有断点及编号 |
| `disable 2` / `enable 2` | `dis 2` / `en 2` | 暂时关闭 / 重新打开 2 号断点 |
| `delete 2` | `d 2` | 删除 2 号断点；不带编号则删除全部断点 |
| `continue` | `c` | 继续运行到下一个断点 |

注意 `*`：`break *phase_01` 停在函数的第一条指令；不带 `*` 的 `break phase_01` 会跳过函数开头保存寄存器、建立栈帧的几条指令，停下时栈已经变化。观察参数时使用带 `*` 的写法。

#### 单步执行

| 命令 | 简写 | 作用 |
| --- | --- | --- |
| `stepi` | `si` | 执行一条指令，遇到 `call` 时进入被调用函数 |
| `nexti` | `ni` | 执行一条指令，遇到 `call` 时把整个调用执行完 |
| `stepi 5` / `nexti 5` | `si 5` / `ni 5` | 连续执行 5 条指令 |
| `advance *0x401234` | `adv *0x401234` | 一直运行到指定地址，常用于跳过一段循环 |
| `finish` | `fin` | 运行到当前函数返回，之后用 `p/u $eax` 查看返回值 |

炸弹没有调试信息，`step`、`next` 无法按源码行执行：在关卡函数中执行 `next` 会一直运行到函数返回。请使用 `si`、`ni`。

#### 查看汇编

| 命令 | 简写 | 作用 |
| --- | --- | --- |
| `disassemble` | `disas` | 查看当前函数的汇编，`=>` 标出下一条要执行的指令 |
| `disassemble phase_01` | `disas phase_01` | 查看指定函数的汇编 |
| `x/10i $pc` | 无 | 从当前位置起显示 10 条指令 |
| `display/i $pc` | `disp/i $pc` | 之后每次停下都自动显示下一条要执行的指令 |

GDB 和 `objdump` 默认使用 AT&T 语法，与教材一致：源操作数在前，目的操作数在后，寄存器带 `%`，立即数带 `$`。

#### 查看寄存器

| 命令 | 简写 | 作用 |
| --- | --- | --- |
| `info registers` | `i r` | 显示全部寄存器；`i r rdi rsi` 只显示指定的几个 |
| `print $rdi` | `p $rdi` | 打印一个寄存器；`$eax` 是 `$rax` 的低 32 位 |
| `print/x $rsi` | `p/x $rsi` | 按十六进制打印；`/u`、`/d`、`/t` 分别按无符号十进制、有符号十进制、二进制打印 |
| `print $rdi + 8` | `p $rdi + 8` | 寄存器可以参与运算，常用于计算地址 |

x86-64 Linux 的调用约定规定，函数的前六个整数或指针参数依次放在 `rdi`、`rsi`、`rdx`、`rcx`、`r8`、`r9`，返回值放在 `rax`（32 位时是 `eax`）。因此在关卡入口处，`rdi` 指向输入的口令字符串，`rsi` 是 challenge；每遇到一次 `call`，都要按那次调用重新判断各寄存器的含义。

#### 查看内存

`x` 命令的写法是 `x/数量格式单位 地址`，`x` 本身没有简写。格式常用 `x`（十六进制）、`u`（无符号十进制）、`d`（有符号十进制）、`s`（字符串）和 `i`（指令）；单位 `b`、`h`、`w`、`g` 分别表示 1、2、4、8 字节。

| 命令 | 作用 |
| --- | --- |
| `x/s $rdi` | 把 `rdi` 指向的内存当作字符串显示 |
| `x/6uw 地址` | 从该地址起读 6 个 4 字节无符号整数，适合查看数组 |
| `x/2gx 地址` | 读 2 个 8 字节值并以十六进制显示，适合查看指针 |
| `x/4wx $rbp-0x20` | 查看栈帧中的局部变量，偏移从汇编中读取 |
| `x/8gx $rsp` | 查看栈顶的 8 个 8 字节值 |

x86-64 是小端序：多字节整数的低位字节存放在低地址。用 `x/8xb` 按字节查看时，要从高地址往低地址读出完整的数。

#### 调用栈

| 命令 | 简写 | 作用 |
| --- | --- | --- |
| `backtrace` | `bt` | 查看调用栈，递归时可以看到每一层 |
| `info frame` | `i f` | 查看当前栈帧，包括返回地址保存的位置和调用者 |

#### 自动显示与分栏

`display` 让 GDB 每次停下都自动打印表达式，例如 `display/i $pc`、`display/x $rax`；`info display` 列出编号，`undisplay 编号` 取消。

`layout asm` 和 `layout regs` 在终端中分栏显示汇编和寄存器，单步时可以同时看到指令和寄存器的变化；`tui disable` 退出分栏。

#### 批处理与命令文件

不进入交互界面也能执行 GDB 命令，适合导出汇编：

```sh
gdb -batch -ex 'disassemble phase_01' ./bomb > phase_01.gdb.txt
```

每次调试都要重复的命令可以写进文件，例如 `phase1.gdb`：

```text
set pagination off
break *phase_01
run --seed 42 --phase 1 < input.txt
display/i $pc
```

然后用 `gdb -x phase1.gdb ./bomb` 启动，GDB 会依次执行文件中的命令。

### tutor 教学

`tutor/` 目录中是一个独立的教学关卡，提供源码和参考解法，不计分，也不需要填写学号或下载个人炸弹。它用上面的 GDB 命令完整演示一次“导出汇编 → 在线调试 → 写出求解函数”的过程，建议在开始正式关卡前做一遍。

在仓库根目录运行：

```sh
make -C tutor test
```

然后按 [tutor 教学](https://github.com/ICS-26Fall-FDU/BombLab/blob/main/tutor/README.md) 的说明逐步操作。根目录的 `make test-tutor` 也可以运行 tutor 教学的测试。

### 实验操作

#### 运行炸弹

```sh
./bomb --seed 42 --phase 1
```

程序显示本关的 challenge，然后等待输入一行口令。口令由若干个无符号十进制整数组成，炸弹不提示个数，需要从汇编中自己找出。`--phase 1` 表示只运行第 1 关；省略这个参数则依次运行六关。也可以把口令写进文件，用重定向输入，例如 `./bomb --seed 42 < answers.txt`。

seed 用来生成各关的 challenge。同一个 seed 和关卡对应同一个 challenge，适合反复调试；换 seed 可以检查另一组输入。分析时先固定 seed，不要每次运行都换。

#### 调试炸弹

!!! tip "提示：口令格式"

    如果不确定口令需要几个整数，不妨看看 `sscanf` 这类输入解析函数，线索也许就在它们的调用附近。

各关入口函数名是 `phase_01` 至 `phase_06`。下面以第 1 关为例，两种方法可以配合使用。

#### 方法 1：离线分析 { .bomblab-method }

导出本关汇编，在编辑器中阅读和标注：

```sh
objdump -d --no-show-raw-insn --disassemble=phase_01 ./bomb > phase_01.S
```

找出函数调用、分支、循环、比较和返回的位置。如果关卡调用了其他函数，也要导出并阅读这些函数，不要只读入口函数。这种方法不运行炸弹，适合先理清计算关系和控制流程。

#### 方法 2：GDB 在线调试 { .bomblab-method }

**准备调试输入。** 关卡会先检查输入格式，不符合时可能提前返回。根据汇编确定本关需要的整数个数后，在 `defuser.c` 对应的函数中把 `count` 改为这个个数，`values` 可以暂时保持 0，然后生成输入：

```sh
make
./defuser --seed 42 --phase 1 > input.txt
```

`input.txt` 是临时的调试输入文件，保存 `defuser` 输出的一行口令，供 GDB 重复读取。`>` 把输出写入文件，`<` 让炸弹从文件读取；文件名可以更换，不需要提交。占位口令只用于观察程序如何处理输入，不表示已经找到正确答案。

**设置断点并运行。**

```text
$ gdb ./bomb
(gdb) break *phase_01
(gdb) run --seed 42 --phase 1 < input.txt
(gdb) x/s $rdi
(gdb) p/x $rsi
```

不想使用文件时，去掉 `< input.txt`，运行后在终端手动输入一行口令即可。

**单步检查。** 用 `ni`、`si` 逐条执行到关键的计算和比较，查看寄存器和内存，核对从汇编中推测的计算关系。

每次修改解法后，重新编译并生成 `input.txt`；换 seed 或关卡时，生成口令和运行炸弹的参数也要一起修改。把确认的计算关系写进对应的 `solve_phase_N`，再按下一小节测试。

#### 测试炸弹

每次修改 `defuser.c` 后先重新编译，再检查同一个 seed 下的口令：

```sh
make
./defuser --seed 42 --phase 1 | ./bomb --seed 42 --phase 1
```

管道左边生成口令，右边检查口令，两边的 seed 和关卡必须一致。通过时会显示 `Phase 1 defused.`。

运行第 1 关的公开测试：

```sh
python3 public_test.py --phase 1
```

模板中的六关求解函数尚未实现，初次测试会出现以下结果，中间省略了其他 seed：

```text
Phase 1 seed 0: FAIL (solver_error)
...
Public tests: 0/10
```

这里的 `solver_error` 来自尚未实现的求解函数。完成本关解法后，重新运行测试；全部公开 seed 通过时会显示 `Public tests: 10/10`。

一次测试全部正式六关：

```sh
make test
```

六关全部公开测试通过时显示 `Public tests: 60/60`，随后检查 Honor：还没有实现 `solve_secret` 时显示 `Secret phase: not attempted`，不算失败。

公开测试通过后，再选一个没有用于调试、也不在公开列表中的 seed 检查，例如：

```sh
./defuser --seed 43 | ./bomb --seed 43
```

一次运行正确不能证明所有输入都正确。还需要检查分支、循环边界和不同的数据起点。

#### GitHub Actions 自测

提交并 push 个人 `bomb` 和解法后，仓库中的 `BombLab public tests` 工作流会自动运行。它分别测试正式六关和选做 Honor，不读取 `config.txt`，不需要学号、Secrets 或登录令牌。

在 GitHub 仓库的 Actions 页面打开本次运行，查看 `Phase 1 public tests` 至 `Phase 6 public tests`。某关失败不会取消其他关，点开任务日志可以查看失败的 seed 和错误类别。

> **Honor 是选做题。** 没有实现 `solve_secret` 时，`Secret phase public tests (optional)` 显示 `not attempted` 并通过；实现后才逐个 seed 检查。这一项失败只表示 Honor 有问题，不代表正式六关没通过，也不会让正式六关扣分，请分别查看六个正式关卡的结果。

Actions 只检查公开测试，不是最终成绩。助教会使用保存的个人炸弹重新编译解法，运行隐藏测试；报告按提交要求检查，不单独计分。

### Honor

一次运行全部六关（不带 `--phase`），并且口令来自管道或文件时，炸弹在六关通过后还会读取第七行；没有第七行时正常结束。在终端或 GDB 中手动输入六行，都不会进入 Honor。

求解接口是 `defuser.h` 中的：

```c
int solve_secret(uint64_t challenge, uint64_t context, char *out, size_t cap);
```

`challenge` 是 Honor 的 challenge，`context` 由前六行口令得到。`./defuser --secret` 会先调用六个正式关卡的求解函数计算 `context`，六关未完成时会直接报错。`solve_secret` 写出完整的第七行，不受正式六关整数格式的限制，但仍须是一行 ASCII 文本，不包含换行。

可以从 `secret_gate`、`secret_copy` 和 `secret_phase` 开始分析。入口和校验方式需要自己观察，不要求覆盖返回地址，也不需要 shellcode。

??? tip "调试提示：用文件输入进入 Honor"

    先用已完成的六关生成前六行，再追加一行试探输入：

    ```sh
    ./defuser --seed 42 > secret.txt
    echo 'test' >> secret.txt
    gdb ./bomb
    ```

    在 GDB 中重定向输入，参数要完整写出：

    ```text
    (gdb) break *secret_gate
    (gdb) run --seed 42 < secret.txt
    ```

    Honor 的 challenge 不会打印，可以在 `secret_gate` 入口按调用约定查看参数寄存器。

??? tip "可参考的仓库材料"

    - [`teaching/07_stack_copy.c`](https://github.com/ICS-26Fall-FDU/BombLab/blob/main/teaching/07_stack_copy.c) 演示同一个结构体内的相邻字段覆盖，可以先对照它观察局部对象的布局。

完成后验证：

```sh
make
{ ./defuser --seed 42; ./defuser --secret --seed 42; } | ./bomb --seed 42
python3 public_test.py --secret
```

只有输出 `Secret phase defused.` 才算 Honor 通过；公开测试全部通过时显示 `Secret public tests: 10/10 (optional)`。`make test` 在六关全部通过后也会自动做这项检查。

公开测试和评分都要求同一个 seed 下的前六关正确。`prerequisite_` 开头的错误表示前六关在该 seed 下尚未通过，不是 Honor 答案本身的错误；评分时某个隐藏 seed 下任一正式关卡出错，该 seed 的 Honor 也不得分。

## 常见问题

### `make bomb` 下载失败

下载程序会自动重试，并在普通下载入口连接失败时尝试 GitHub 官方 API。在线下载会校验 SHA-256，通过后才安装炸弹，不需要额外填写令牌。

如果仍然失败，可以在浏览器中打开 [炸弹下载页面](https://github.com/ICS-26Fall-FDU/BombLab/releases/tag/bombs)，下载自己的 `bomb-<学号>.tar.gz`，再导入：

```sh
python3 fetch_bomb.py --file bomb-243xxxx0175.tar.gz
make
```

把文件名换成自己的附件路径，并确保配置中的学号正确。已有 `bomb` 时无需重新下载；只有丢失或删除炸弹后，才需要重新填写学号领取。

如果助教发布了新版炸弹，想更新终端剧情或展示，请先备份自己的解法，在 `config.txt` 中重新填写学号，直接运行 `python3 fetch_bomb.py`。它只替换 `bomb` 和 `package.json`，不会覆盖 `defuser.c`；领取后可再次清空学号。已有 `bomb` 时，`make bomb` 不会主动更新。

### 炸弹无法执行

先检查 `uname -sm` 是否为 `Linux x86_64`。如果出现 `Permission denied`，执行 `chmod +x bomb`；如果出现 `Exec format error` 或 `cannot execute binary file`，检查是否正在 Windows、macOS 或 ARM 环境中运行，换到符合要求的 Linux 环境。

### 公开测试失败

| 错误类别 | 含义及检查方向 |
| --- | --- |
| `solver_error` | 解法程序返回非零或异常退出；先检查是否已实现、是否返回 0，以及 stderr 中的错误提示 |
| `format_error` | 输出不是规定的一行口令；检查换行、编码和是否混入调试信息 |
| `output_limit` | 输出过长；检查是否向 stdout 输出了日志或重复内容 |
| `wrong_answer` | 口令未通过炸弹检查；检查字段数、计算过程、运算宽度及两边的 seed 和关卡 |
| `timeout` | 解法未在规定时间内结束；检查死循环、递归和算法耗时 |

编译失败时先处理 GCC 的错误，不要直接继续运行旧的 `defuser`。单个 seed 失败时，可以用同一个 seed 和关卡复现，再用 GDB 检查。

### Actions 提示 `Personal bomb is missing`

模板默认忽略下载和编译产物，普通的 `git add` 不会添加炸弹。执行 `git add -f bomb`，然后 commit、push。Actions 会恢复炸弹的可执行权限，不需要为此保留学号配置。

### 修改配置能换题吗

不能。配置只用于下载，正式评分按学号使用助教保存的对应炸弹。误领他人的炸弹时，请先保留自己的解法文件，删除错误的 `bomb`，用自己的学号重新领取。

### 其他问题

遇到问题时，在课程讨论区提问或联系本实验助教，联系方式见课程公告。请说明运行环境、使用的命令、报错信息，以及已经尝试过的方法；涉及关卡解法的内容不要公开贴出完整代码。

## 提交 { .text-bold }

**提交截止日期：2026 年 10 月 31 日。** 记得在截止前将仓库链接提交到 E-Learning，并将解法、报告和 `final` 标签推送到 GitHub。具体截止时刻以 E-Learning 作业页面为准。

### 内容要求

仓库根目录需要有：

| 文件 | 要求 |
| --- | --- |
| `defuser.c` | 正式关卡解法；Honor 解法也写在这个文件中 |
| `report.md` 或 `report.pdf` | 实验报告，任选一种格式，不提交 `.doc` 或 `.docx` |
| `bomb` | 个人炸弹，用于 Actions 公开自测 |

姓名、学号不作报告或提交格式要求，可自行选择是否填写。`config.txt` 和 `package.json` 不要求提交。不要提交解法可执行文件 `defuser`、tutor 教学的编译产物、整份反汇编或调试日志。

助教评分不使用上传的 `bomb`，缺少它不扣提交格式分，但 Actions 无法运行。请不要把 `password.txt` 当成解法提交。

报告需要包含：

- 每个已完成关卡的关键汇编、恢复出的计算关系和推理过程；可以配数据流图或控制流图（不计分数，不强制要求，只是为了更好地展现思路）。
- 每个已完成关卡至少一次使用公开列表以外的 seed 验证的记录。
- 已完成关卡的运行结果截图；尚未完成时，如实说明进展和遇到的问题。
- 引用的资料，以及课程允许的辅助工具的用途。

报告不单独计分，但仍需提交。简洁清楚即可，不需要逐行翻译全部汇编。对实验的意见和建议可以附在最后。

Markdown 报告中的图片放在仓库的 `images/` 或 `assets/` 目录，使用相对路径，例如 `![运行结果](images/result.png)`。图片需要一起提交；提交后在 GitHub 中打开报告，确认图片能显示。

### 上传仓库链接和代码

**先在 E-Learning 本实验作业页面提交自己的 GitHub 仓库链接。** 确保仓库公开。

然后在仓库根目录执行。以下使用 PDF 报告，Markdown 报告将 `report.pdf` 换成 `report.md`，并添加引用的图片目录：

```sh
git add -f bomb
git add defuser.c report.pdf
git commit -m "submit lab2"
git tag final
git push
git push origin final
```

> **普通的 `git push` 不会自动推送新标签。** 最后一条命令不能省略。到 GitHub 的 Tags 页面检查 `final` 是否存在、是否指向本次提交；同时检查报告和最新 Actions 结果。

截止前更新作业时，重新提交并移动 `final` 标签：

```sh
git add defuser.c report.pdf
git commit -m "update lab2"
git tag -f final
git push
git push -f origin final
```

使用 Markdown 报告时，同样替换文件名并提交新增图片。助教按截止时刻收集到的 `final` 提交评分，不是按你本地尚未 push 的内容评分。

### 评分规则

| 项目 | 分值 |
| --- | ---: |
| 提交格式：有 `defuser.c` 和报告、已推送 `final` 标签 | 2 |
| 正式六关，每关 18 分 | 108 |
| 实验报告（必交，不单独计分） | 0 |
| 基础分合计 | 110 |
| Honor（选做），额外加分 | +10 |
| 含 Honor 的最高总分 | 120 |

每个正式关卡使用 128 个隐藏 challenge，按通过比例计分，例如通过一半测试得 9 分。Honor 使用 128 个隐藏 seed，同样按通过比例计分。公开测试不单独计分，只针对公开 seed 写答案表不能完成实验。

实验报告的要求见上面的“内容要求”；未提交报告不能获得格式分。报告检查与抽查安排以课程通知为准。未完全通关时，可以记录已经确认的分析过程和遇到的问题。

## 协作与学术诚信

可以讨论 GDB 的使用、汇编指令和分析方法，不要交换各关的具体计算逻辑、解法代码或报告，也不要复制他人的实现。

引用资料需要在报告中列出。AI 工具的允许范围以课程公告为准；使用允许的工具时说明用途，并能解释自己的分析和代码，不要将工具生成的完整解法当作自己的分析提交。

## 参考资料与致谢

- GDB 官方文档：<https://www.sourceware.org/gdb/documentation/>
- CS:APP x86-64 GDB 速查表：<http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf>
- Beej's Quick Guide to GDB：<https://beej.us/guide/bggdb/>
- 知乎专栏文章（延伸阅读）：<https://zhuanlan.zhihu.com/p/1961833750619988266>
- `objdump` 手册：在终端运行 `man objdump`。

感谢课程助教团队，以及上述资料的作者与维护者。
