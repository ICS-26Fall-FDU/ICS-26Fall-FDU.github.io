---
title: "Lab1：DataLab"
---

# Lab1: DataLab

## 实验简介

本实验是 CSAPP 第一章配套实验，目的是加深同学们对整数和浮点数二进制表示的认识。同学们需要解出若干程序谜题，编写代码并通过正确性测试，最后提交代码和报告。希望同学们多加思考，在解题过程中能学到的远不止二进制本身，还能加深对位运算的理解，以及学到一些算法知识。

本实验的满分为 110 分。迟交 5 天内，最终得分为得分的 80%；迟交 5 天以上，最终得分为得分的 60%。截止日期后一天和后六天时 TA 会运行分数计算脚本。

受人力所限，对于每位同学，我们会从本学期的所有 Lab 的实验报告中抽取一份打分并作为实验报告分数。

## 部署实验环境

### 领取作业并克隆仓库到本地

与 [Lab0](/labs/lab0-git-lab) 一样，请使用模板仓库 [DataLab](https://github.com/ICS-26Fall-FDU/DataLab) 创建自己的仓库。

获取仓库后克隆到本地：

```shell
git clone <仓库地址>  # 将 <仓库地址> 替换为上一步得到的仓库链接
cd <仓库名>
```

### 环境要求

在支持 32 位编译的 x86-64 Linux 环境中完成实验可以获得完整评测体验，例如课程服务器、Ubuntu 虚拟机或 WSL。仓库中的 `dlc` 是 Linux ELF 可执行文件，不能直接在 macOS 或 Windows 上运行。

若在不支持评测的系统如 M 系列芯片 MacOS 完成实验，也可以通过 Github Workflow 线上查看评测结果。完成提交和推送后，在仓库的 Actions 页面点入 Workflow 详情后，可以通过 Annotations 查看总分，也可以点击 run-autograding-tests 进入每个测试的详细流程检查，查看 Autograding Reporter 项（方法不稳定，可能需要多次刷新）或在右侧设置项中下载日志查看测试具体情况。

请注意**不要**修改 .github 文件夹内预设的自动化测试内容。我们在检查分数时会校验这一路径下文件的完整性。如果这一目录下的文件被修改，你将会拿到 0 分。

### 面向 32 位 Linux ELF 目标的编译环境配置

执行：

```shell
sudo apt-get update
sudo apt-get install -y gcc make gcc-multilib libc6-dev-i386 python3
```

如果编译时提示缺少 `bits/libc-header-start.h`、`-lgcc` 或其他 32 位库，通常是 `gcc-multilib` 或 `libc6-dev-i386` 未安装完整。

### 确认实验文件能正常构建

键入 `ls`，你应当看到如下文件：

```text
Driverhdrs.pm  Driverlib.pm  Makefile  README.md  bits.c  bits.h  btest.c
btest.h  check_ops.py  decl.c  dlc  driver.pl  fshow.c  ishow.c  test.sh  tests.c
```

在终端中依次执行下述指令，以生成可执行文件并执行：

```shell
make clean
make all
./btest
```

`make all` 会生成 `btest`、`ishow`、`fshow` 三个可执行文件。如果过程顺利，`./btest` 的最后一行会输出 `Total points: 0/110` 。

如果遇到 `./check_ops.py: Permission denied` ，说明当前文件没有执行权限，执行：

```shell
chmod +x check_ops.py dlc
```

!!! note

    `btest` 不会在源文件修改后自动重编译。每次改完 `bits.c` 都要重新执行 `make clean && make all` ，否则测试的可能仍是旧代码。

## 主要文件

| 文件 | 用途 |
|---|---|
| `README.md` | 实验说明，包含每道题的完整规则 |
| `bits.c` | 唯一需要填写的代码文件；文件开头包含完整编码规则 |
| `bits.h` | 函数声明，不能修改 |
| `check_ops.py` | 当前题目的规则与操作数检查入口 |
| `dlc` | `check_ops.py` 内部调用的 DataLab 规则检查器 |
| `decl.c`、`tests.c`、`btest.c` | 题目参数范围、参考行为与正确性测试 |
| `Makefile` | 构建 `btest`、`ishow` 和 `fshow` |
| `test.sh` | 一次执行构建、规则检查和完整测试 |
| `ishow`、`fshow` | 编译后生成的整数和浮点位表示辅助工具 |

## 实验提示与说明

### 如何入手

推荐阅读顺序：本文档 > `README.md` > `bits.c` 中每个函数上方的注释。

你只需要修改 `bits.c` 中 P1–P19 的函数体。**不要修改函数名、参数、返回类型、测试程序或评分配置。**

`README.md` 与 `bits.c` 的注释中对实验文件做了较为详细的介绍，并给出了逐题的函数签名、输入约束、合法运算符、最大操作数和分值。上面两个文件请务必仔细阅读。

注意到，每一个谜题包含了如下信息：

- 能使用的运算符。
- 能使用的运算符总数量。
- 能使用的常数的值域范围。
- 变量类型与输入参数的取值范围。
- 能否使用控制语句（如 `if` ）等。

## 测试

除了 `bits.c` ，你不应该编辑任何其余文件。

### 编译

每次修改 `bits.c` 后重新编译：

```shell
make clean
make all
```

### 检查运算符和操作数

```shell
./check_ops.py bits.c
```

输出会列出每道题的“当前操作数/最大操作数”。最后出现以下内容才表示规则检查通过：

```text
All 19 functions passed operator checks.
```

!!! warning

    **不要直接使用 `./dlc bits.c` 检查整份文件。** 仓库自带的 `dlc` 内置了旧函数名，`check_ops.py` 会为新题逐题选择同规则的代理函数名，再调用 `dlc` 检查实际函数体。代理过程不会放宽合法运算符，只解决旧二进制不认识新函数名的问题。

规则检查通过只说明代码写法合法，不表示结果正确。

### 测试正确性

```shell
./btest
```

`btest` 执行时会给出每个谜题（函数）是否通过测试（未通过时会给出测试数据），并且会计算你的最终得分。每道题的 `Errors` 都应为 0，满分为 `110/110` 。

调试单题时不必反复全量测试，可以指定函数名：

```shell
./btest -f roundEvenPow2
```

可用 `-1`、`-2`、`-3` 指定前三个参数：

```shell
./btest -f roundEvenPow2 -1 5 -2 1
./btest -f copyByteWithin -1 0x11223344 -2 0 -3 2
```

失败时会显示输入、实际结果和期望结果。

### 一键流程

也可以一次执行全部流程：

```shell
./test.sh
```

`test.sh` 只有在构建成功、规则检查通过并取得 `110/110` 时才返回状态码 0。在尚未完成全部题目时返回非零是正常现象，调试时应优先使用单题检查。

### 辅助工具

你可以利用 `./ishow` 和 `./fshow` 来帮助你调试（用法见 `README.md` ）：

```shell
./ishow 27
./ishow 0x80000000
./fshow 0x3f800000
./fshow 0x80000000
```

`ishow` 会显示十六进制、有符号和无符号解释；`fshow` 会拆解符号位、阶码和尾数。

## 常见问题

### `./check_ops.py: Permission denied`

终端中输入以下指令（或前面加 sudo 的版本）：

```shell
chmod +x check_ops.py dlc
```

（小技巧：`chmod 777` 可以赋予所有用户一个文件的读、写、执行权限～Linux 文件权限由三个分别代表三类用户权限的八进制数表示，其中每个二进制位从高到低分别代表读、写、执行，每位全设为 `1` 则变成 `777`。）

### `./dlc: cannot execute binary file`

当前环境不是兼容的 x86-64 Linux，请切换到课程服务器、Linux 虚拟机或合适的 WSL 环境。

### 编译提示缺少 32 位库

```shell
sudo apt-get install -y gcc-multilib libc6-dev-i386
make clean
make all
```

### 规则检查通过但 `btest` 失败

检查器只验证语法、运算符和操作数数量；`btest` 才验证结果。根据失败输入判断问题属于符号、溢出、移位、舍入还是浮点特殊值处理。

### `btest` 通过但规则检查失败

算法结果可能正确，但使用了禁用语法、常量或过多操作数，仍不能得分。按 `check_ops.py` 给出的函数定位问题。

### `test.sh` 在未完成实验时失败

这是预期行为，因为脚本要求满分。使用 `./btest -f 函数名` 调试单题。

## 提交

截止时间：10 月 9 日 23:59。逾期将扣除部分分数。

### 内容要求

你需要提交至少两份文件，包含你的 `bits.c` 和一份实验报告。

实验报告应该包含以下内容：

- 实验标题，你的姓名，学号。
- 你在终端中执行 `./check_ops.py bits.c` 后的截图。
- 你在终端中执行 `./btest` 后的截图。
- 描述你实现每个函数的思路。`bits.c` 中不要求给自己的代码写注释（写了也无妨）
- 如果有，请务必在报告中列出引用的内容以及参考的资料。
- 对本实验的感受（可选）。
- 对助教们的建议（可选）。

### 格式要求

可提交 `.md` 文件或者 `.pdf` 文件。请勿提交 `.doc` 或 `.docx` 文件。  
（如果提交 `.md` 文件，请确保助教能同时看到你报告中的截图！）

!!! tip

    在你的 Markdown 报告中插入图片时，请将图片文件放在仓库的 `images/` 或 `assets/` 文件夹下，并使用 **相对路径**（如 `images/screenshot.png`）插入图片。常见支持的图片格式有 PNG、JPG 和 GIF。插入图片的 Markdown 语法示例：`![截图说明](images/screenshot.png)`。上传报告时，请确保图片文件已一并上传到 GitHub。如果你不确定这一步是否正确，请进入你的远程仓库预览你的报告。

### 上传

首先，在 E-Learning 平台上提交你根据模板仓库创建的个人仓库链接。

打开终端，**在实验仓库路径下** 执行以下指令，或使用图形化界面操作：

```shell
# 提交当前文件夹下的所有更改到暂存区
git add -A

# 将暂存区的所有更改提交到本地仓库
git commit -m "xxx(可以是你的提交注释)"

# 将本地仓库推送到远程
git push
```

!!! note

    提交前请自查：没有提交 `btest`、`ishow`、`fshow`、`*.o` 等编译产物。

本实验使用 GitHub Actions 自动评分：每次推送都会对 19 道题逐题检查运算符规则与正确性，总分 110 分。

### 评分规则

- 实验报告要求简洁清晰，不必追求字数，描述清楚思路即可。
- 对每道谜题请先自行思考，不要立即使用搜索引擎或 AI 直接生成答案。
- 可以查阅 C 语言、补码、位运算和 IEEE 754 资料，但不要复制他人的实现。
- 严禁抄袭其它人的代码，一旦发现，零分处理。

## 参考资料

- [CMU 原版 Lab](https://csapp.cs.cmu.edu/3e/labs.html)
- [实验入门手册](./manual.md)
- 本文档编写时参考了 25 年的实验文档 。

!!! info "本 Lab 负责助教"

    - [张林涛](mailto:lintaozhang25@m.fudan.edu.cn)

!!! info "特别鸣谢"

    - 25 年秋学期 ICS 助教团队
