# simple-sched-lab C 语言开发规范与代码审查指南

本文件是 **simple-sched-lab**（进程调度观测实验）C 代码的开发约定与代码审查依据。
实验以 C 为实现基础，同时用独立的 C++17 树学习系统编程。两套代码由 CMake 隔离为 `sched_c` 与 `sched_cpp`，互不混编。C++ 代码以 `code_review_cpp.md` 为准；C 代码、C 头文件以及审查意见以本文为准。
同一改动里两种语言并存时，按文件扩展名分别套用对应规范，不要用其中一份去覆盖另一份。

## 本实验要做什么（审查时对照）

同时运行一个或多个**一味消耗 CPU 时间**的进程，采集：

- 某一时间点运行在逻辑 CPU 上的是哪一个进程
- 每个进程的运行进度

据此核对本实验对调度器（分时、抢占、多进程共享 CPU）的描述是否正确。

命令行：

```text
./sched_c n total resol
```

| 参数 | 含义 |
| --- | --- |
| `n` | 同时运行的进程数量 |
| `total` | 每个进程消耗的 **CPU 时间**（毫秒），达到后该进程结束 |
| `resol` | 采集统计信息的间隔（毫秒） |

行为约定：

1. 启动 `n` 个进程同时运行；全部结束后父进程再退出。
2. 每个进程在消耗 `total` 毫秒 CPU 时间后结束（不是 `sleep(total)`，也不是墙钟到点就退出）。
3. 每 `resol` 毫秒记录一次：① 进程唯一 ID（`0` ~ `n-1`）；② 从**程序开始运行**到该采样点经过的时间（毫秒，墙钟）；③ 进度（%）。
4. 全部结束后，把所有统计信息用**制表符分隔、逐行**输出。

## 1. 规范来源与优先级

1. **Linux Kernel Coding Style** 为代码风格的权威来源：
   [https://docs.kernel.org/process/coding-style.html](https://docs.kernel.org/process/coding-style.html)
2. 本文是该规范在本项目中的**落地摘要与审查清单**。内核文档面向内核树，本文将其映射到用户态 C，并补充本实验的开发约定。
3. 冲突处理：
   - 缩进、花括号、命名、函数长度、goto 清理、宏与注释等风格细节以 Linux Kernel Coding Style 为准，**下列项目例外除外**。
   - 构建、目录、进程/时钟/采样输出等项目约定以本文「项目约定」与「项目专项」章节为准。
   - 内核特有机制（`kmalloc`、`printk`、Kconfig、`EXPORT_SYMBOL`、`BUG()` 等）不直接搬进用户态；用第 1.1 节的对应物。
   - 审查时发现规范未覆盖的问题，以可读性、正确性和一致性为准，并回写本文。

### 1.1 相对内核规范的项目映射 / 例外

本项目是用户态实验，不是内核模块。风格跟内核走，API 与运行时用用户态对应物：

| 项 | Linux Kernel | 本项目 |
| --- | --- | --- |
| 运行环境 | 内核 | **用户态** |
| 语言 | 内核 gnu11 | **GNU C11**（`CMAKE_C_STANDARD=11`），禁止 C23 及编译器私有扩展 |
| 定宽整数 | `u8` / `u32` / `u64` 等 | **`<stdint.h>`**：`uint8_t` / `uint32_t` / `uint64_t` 等 |
| 布尔 | 内核 `bool` | **`<stdbool.h>`**：`bool` / `true` / `false` |
| 内存分配 | `kmalloc` / `kzalloc` / `vmalloc` | **`malloc` / `calloc` / `realloc` / `free`** |
| 日志 | `printk` / `dev_err` / `pr_*` | **`fprintf(stderr, ...)`** 或项目统一日志宏；系统调用失败可辅以 `perror` |
| 错误码 | 负 errno、`ERR_PTR` | 动作型函数返回 **0 成功 / 负 errno 失败**；指针型失败返回 **`NULL`** |
| 导出符号 | `EXPORT_SYMBOL` | 不适用；对外符号用 `sched_lab_` 前缀，文件内符号 `static` |
| 配置裁剪 | Kconfig / `IS_ENABLED` | 不引入 Kconfig；编译期开关用 CMake option + 头文件里的 stub |
| 断言 / 崩溃 | `WARN*` / `BUG*` / `panic` | **禁止**用 `abort` / `assert` 处理可预期失败；`assert` 只用于内部不变量 |
| 缩进 / 花括号 / 行宽 | Tab 8、K&R、80 列 | **与内核一致，无例外** |

除此以外不另起炉灶。内核文档中与用户态无关的章节（Kconfig 缩进、`GFP_*`、内联汇编惯例、`do not crash the kernel` 的 `panic_on_warn` 等）不作为本项目审查条款。

## 2. 开发约定

### 2.1 语言与工具链

| 项 | 约定 |
| --- | --- |
| 语言 | GNU C11（`CMAKE_C_STANDARD=11`），禁止 C23 及编译器私有扩展 |
| 编译器 | GCC 或 Clang，以 Linux GNU 工具链为主 |
| 构建 | CMake ≥ 3.10，使用 `CMakePresets.json` 中的 preset |
| 编译数据库 | `compile_commands.json`（供 clangd 使用） |
| 格式化 | `clang-format`，Linux / K&R 风格：Tab 缩进、宽度 8、列宽 80 |
| 风格检查 | 内核 `checkpatch.pl` 仅作参考（它假设内核树）；审查不以工具通过为唯一标准 |

推荐 `clang-format`（放在 `sched_c/.clang-format`，只约束 C 树）：

```yaml
BasedOnStyle: LLVM
IndentWidth: 8
TabWidth: 8
UseTab: Always
BreakBeforeBraces: Linux
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false
IndentCaseLabels: false
ColumnLimit: 80
SortIncludes: false
```

CMake 用 option 隔离两套代码：`SIMPLE_SCHED_BUILD_C` / `SIMPLE_SCHED_BUILD_CPP`。preset：

| preset | 目标 |
| --- | --- |
| `simple-lab` | 同时构建 `sched_c` 与 `sched_cpp` |
| `sched-c` | 只构建 C |
| `sched-cpp` | 只构建 C++17 |

```bash
cmake --preset simple-lab
cmake --build --preset simple-lab
```

只编一侧：

```bash
cmake --preset sched-c
cmake --build --preset sched-c
```

本地提交前至少完成：

1. 代码可编译。打开 `-Wall -Wextra`；warning 要看、要处理，但**不要** `-Werror`（警告不导致构建失败）。
2. `clang-format` 已格式化本次改动。
3. 手动走通本次改动对应的实验路径（至少：`n=1` 与 `n>1` 各跑一遍，检查输出列与进度）。

### 2.2 目录与文件

两套代码分目录隔离，不要把 C 和 C++ 源文件放进同一个 target：

```text
simple-sched-lab/
  CMakeLists.txt
  CMakePresets.json
  docs/
  sched_c/              # 目标 sched_c，GNU C11
    CMakeLists.txt
    .clang-format
    main.c
  sched_cpp/            # 目标 sched_cpp，C++17
    CMakeLists.txt
    .clang-format
    main.cpp
```

随着实现展开，在**各自目录内**按职责拆分，而不是把 fork、计时、采样缓冲、输出全堆进一个 `main`，也不要跨目录互相 `#include`。

文件命名：

- 源文件 `.c`，头文件 `.h`。本项目统一使用**小写 + 下划线**：`busy_loop.c`、`sample_log.h`、`time_util.c`。
- 测试文件：`foo_test.c`。
- 头文件与实现成对出现；仅 `main.c`、测试文件可以没有对应头文件。
- 不要使用已在系统路径中常见的名字（如 `time.h`、`sched.h`），避免与系统头冲突。

头文件必须有 include guard，按仓库路径生成，例如 `sched_c/sample_log.h`：

```c
#ifndef SCHED_LAB_SAMPLE_LOG_H
#define SCHED_LAB_SAMPLE_LOG_H
/* ... */
#endif /* SCHED_LAB_SAMPLE_LOG_H */
```

被 C++ 包含的对外 C 头必须用 `extern "C"` 包起来：

```c
#ifdef __cplusplus
extern "C" {
#endif

/* 公开 C API */

#ifdef __cplusplus
}
#endif
```

禁止在头文件里放需要链接的对象定义（非 `static inline` 的函数体、非 `static const` 的可变全局量）。

### 2.3 链接可见性

- 只在本 `.c` 使用的函数、全局变量一律 `static`。
- 跨文件符号使用 `sched_lab_` 前缀，例如 `sched_lab_run_child()`。
- 不在头文件中写 `extern` 函数声明（C 里多余，也会拉长行）；变量的 `extern` 声明可以保留。
- 禁止文件作用域的 `typedef` 指针（见 3.5）；禁止隐式 `int`。

### 2.4 Git 与审查节奏

项目目前可以没有远程仓库，但本地改动仍按可审查单元组织：

- **一次提交只做一件事**：功能、重构、格式化不要混在同一提交。
- 提交说明写清「为什么」，而不是罗列改了哪些文件。
- 审查以 diff 为单位：先看设计与接口，再看实现与风格。
- 实验性简化（例如未处理 `n` 过大、未绑核）可以先合入，但必须在注释或 `docs/` 中标明限制。

### 2.5 错误处理策略

本项目采用内核式返回值约定，而不是 `setjmp` 或把所有失败都 `exit`：

- **动作 / 命令型函数**（`open`、`run`、`init`、`wait`）：返回 `0` 成功，失败返回负的 `errno` 值（如 `-errno` 或 `-ENOMEM`）。
- **谓词型函数**（`is_*`、`has_*`、`*_ok`）：返回 `bool` 或 `true` / `false`，不要返回负错误码。
- **返回指针的函数**：成功返回有效指针，失败返回 `NULL`。不要把错误码编码进指针。
- `fork`、`waitpid`、`clock_gettime`、管道/`write` 等失败必须检查，并保证已经获取的资源在所有路径上释放、已经创建的子进程被回收。
- 多出口且需要清理时，用 **goto 集中清理**（见 3.7），不要复制粘贴一串 `close` / `free` / `waitpid`。
- `assert` 只用于「不可能发生」的内部不变量，不用于可预期的参数错误、`fork` 失败、时钟不可用。
- 不要在库路径上 `exit()` / `abort()`；`main` 里报告错误后以非零状态退出是可以的。

## 3. 代码规范摘要（Linux Kernel Coding Style）

以下为审查时必须核对的要点。细节以官方文档为准。

### 3.1 缩进与折行

- **缩进使用 Tab，显示宽度 8**。注释、文档和字符串对齐可以用空格，但缩进层级本身禁止空格代替 Tab。
- 超过约 3 层缩进应视为函数过深，优先拆分，而不是把行宽继续往右推。
- `switch` 与 `case` 对齐，不要把 `case` 再缩进一层：

```c
switch (suffix) {
case 'G':
case 'g':
	mem <<= 30;
	break;
case 'K':
case 'k':
	mem <<= 10;
	fallthrough;
default:
	break;
}
```

- 一行一条语句；禁止用逗号运算符把多条语句挤进 `if` 里来逃避花括号。
- 不要在一行里做多次赋值。不要在行尾留空白。
- **每行最多 80 列**。超长语句拆成更短的后续行，后续行明显靠右；函数参数列表可对齐到开括号。
- **不要拆用户可见字符串**（日志、帮助文本），以便 `grep`。include、头保护和无法折行的 URL 可以超长。

### 3.2 花括号与空格

非函数语句块（`if` / `switch` / `for` / `while` / `do`）采用 K&R：开括号在行尾。**函数**的开括号独占下一行：

```c
int sched_lab_parse_args(int argc, char **argv)
{
	if (argc != 4) {
		fprintf(stderr, "usage: simple-sched-lab n total resol\n");
		return -EINVAL;
	}
	return 0;
}
```

`do-while` 的 `while`、`if-else` 的 `else` 跟在闭括号同一行：

```c
if (x == y) {
	...
} else if (x > y) {
	...
} else {
	...
}
```

单条语句的 `if` / `else` / 循环**不要**无故加大括号：

```c
if (condition)
	action();
```

但只要有一个分支是复合语句，两个分支都加大括号。循环体里若再套 `if`，循环本身也加大括号。

空格：

- 关键字 `if` / `switch` / `case` / `for` / `do` / `while` 之后有空格。
- `sizeof` / `typeof` / `alignof` / `__attribute__` 之后无空格：`sizeof(struct sample)`。
- 括号内侧无空格：禁止 `sizeof( struct sample )`。
- **`*` 贴变量名 / 函数名，不贴类型**：`char *linux_banner;`、`char *match_strdup(substring_t *s);`（这与本项目 C++ 规范相反，C 文件必须按内核风格）。
- 二元 / 三元运算符两侧有空格；一元运算符、前后缀 `++` `--`、`.` / `->` 周围无空格。

### 3.3 头文件与 include

- 头文件必须自包含：能单独编译，包含自己用到的所有头。
- Include What You Use：用到的符号在本文件直接 `#include`，不要依赖传递包含。
- `.c` 的第一个 include 是对应头文件，以验证头文件自包含。
- Include 顺序（组与组之间空一行）：

  1. 对应头文件（`foo.c` 先包含 `foo.h`）
  2. C 系统 / 标准库头（`<unistd.h>`、`<stdint.h>` 等）
  3. 第三方库头（本实验通常没有）
  4. 本项目头文件

  系统 / 标准库 / 第三方用 `<>`；本项目头用 `""`，路径相对源码根，禁止 `../`。

```c
#include "sched_lab/sample_log.h"

#include <errno.h>
#include <stdint.h>
#include <sys/wait.h>
#include <unistd.h>
```

- 优先在头文件里用 `#ifdef` 提供空操作 stub，`.c` 里无条件调用，避免把业务逻辑淹没在预处理条件中（见 3.13）。

### 3.4 命名

C 是斯巴达式语言，不要 PascalCase / 驼峰，也不要匈牙利命名。

| 类别 | 规则 | 示例 |
| --- | --- | --- |
| 文件 | 小写下划线 + `.c` / `.h` | `sample_log.c` |
| 全局函数 / 全局变量 | 描述性 snake_case，带 `sched_lab_` 前缀 | `sched_lab_run_child()` |
| 文件内函数 / 变量 | `static` + snake_case | `parse_args()` |
| 局部变量 | 短而足够：循环用 `i` / `n`，临时值用 `tmp` | `pid`、`ret`、`i` |
| 结构体标签 | snake_case，不 typedef 掉 | `struct sched_lab_sample` |
| 结构体成员 | snake_case | `id`、`elapsed_ms`、`progress` |
| 枚举标签与常量宏 | 全大写 + 下划线 | `SCHED_LAB_MAX_CHILDREN` |
| 函数式宏 | 可用小写，但优先 `static inline` | `min()` 这类才像函数 |
| 头保护宏 | 全大写，对应路径 | `SCHED_LAB_SAMPLE_LOG_H` |

- 全局名必须能从名字看出职责；`cntprc()` 这种删字母缩写不合格。
- 局部名过长通常说明函数太大（见 3.6）。
- 调度/进程领域通用词（CPU、PID、fork、wait）可以使用。
- 包容性用语：新代码避免 master/slave、blacklist/whitelist，改用 primary/secondary、blocklist/allowlist 等。

### 3.5 Typedef

**默认不要 typedef 结构体和指针。** `vps_t a;` 看不出 `a` 是什么；写成 `struct virtual_container *a;` 才清楚。

允许 typedef 的情况（须对得上其中一条）：

1. 完全不透明、只能通过访问器操作的对象。
2. 宽度会随配置变化、需要挡住 `int` / `long` 混淆的整数抽象——本项目用 `stdint.h` 即可，不要再包一层 `typedef unsigned long myflags_t`。
3. 与用户态共享、或对接内核 UAPI 时必须使用的既有类型。

可直接访问字段的 struct、以及指针类型，禁止再 typedef。本项目对外类型写成 `struct sched_lab_sample`，而不是 `sched_lab_sample_t`。

### 3.6 函数

- 函数短小、只做一件事。经验上应能放进一到两屏（约 80×24）；越复杂、缩进越深，越要短。
- 局部变量大致 **5–10 个**为上限，再多就拆 helper。
- 源文件中函数之间空一行。
- 原型必须写出参数名，不要只写类型。
- 原型元素顺序：存储类（`static`）→ 属性（如 `__attribute__((unused))`）→ 返回类型 → 函数名 → 参数 → 其余属性。
- 禁止无原型的旧式声明；参数表为空时写 `void`：`int sched_lab_init(void);`。
- `inline` 克制：超过约 3 行不要随手标 `inline`。`static` 且仅一处调用时让编译器自己决定，不要提前写 `inline`（否则第二处调用时还得删）。函数式宏优先改成 `static inline`。
- 参数解析、子进程忙等与采样、父进程回收与打印应拆开，不要揉成一个巨大的 `main`。

### 3.7 集中清理（goto）

有多处失败且需要释放资源时，用 goto 跳到函数尾部的清理标签；没有清理工作则直接 `return`。

- 标签名说明**要做的事**：`out_free_samples:`、`err_close_pipe:`、`err_reap_children:`。禁止 `err1:` / `err2:`。
- 按资源获取的**反序**设置多个标签，避免「`foo` 仍是 `NULL` 却 `free(foo->bar)`」这类 one-err 缺陷。
- 理想情况下应能模拟失败，走遍所有出口。`fork` 失败时尤其要回收已经拉起来的孩子。

```c
int sched_lab_spawn_children(int n, pid_t **out_pids)
{
	int i, ret;
	pid_t *pids;

	pids = calloc(n, sizeof(*pids));
	if (!pids)
		return -ENOMEM;

	for (i = 0; i < n; i++) {
		pids[i] = fork();
		if (pids[i] < 0) {
			ret = -errno;
			goto err_reap;
		}
		if (pids[i] == 0) {
			/* child path */
			return 0;
		}
	}

	*out_pids = pids;
	return 0;

err_reap:
	while (i-- > 0) {
		if (pids[i] > 0)
			waitpid(pids[i], NULL, 0);
	}
	free(pids);
	return ret;
}
```

（上例仅说明清理顺序；真实实现还需区分父子路径，避免子进程误走父进程清理。）

### 3.8 注释

- 注释写**做什么 / 为什么**，不要复述怎么做。能靠代码本身说清的，不要堆注释。
- 优先在函数头部注释；函数体内部只对特别巧妙或特别难看的地方留短注。函数复杂到必须分段注释时，先拆函数。
- 多行注释用内核块注释：

```c
/*
 * Preferred multi-line comment style.
 *
 * Describe non-obvious constraints here.
 */
```

- 公开函数注明：用途、所有权（谁分配谁释放）、失败语义、进程约定（若有）。
- 时钟选择（为何用 `CLOCK_PROCESS_CPUTIME_ID` 而不是墙钟）、采样为何不能 `sleep`、子进程为何不得直接写 stdout 等**非显然约束**，必须在实现处注释。
- 数据声明一行一个成员，右侧可以跟短注释。
- `TODO` 格式：`/* TODO(owner): 要做什么及原因 */`。
- 注释、标识符和提交说明：本项目文档可用中文；代码标识符使用英文。不要中英混杂命名。
- **禁止**在源文件里嵌入 editor modeline（`vim:set`、`/* -*- mode: c -*- */`、Local Variables 等）。

### 3.9 宏、枚举与条件编译

- 相关常量优先 `enum`；孤立常量用全大写 `#define`，表达式必须加括号。
- 多语句宏包在 `do { ... } while (0)` 里。
- 优先 `static inline` 函数，而不是函数式宏。
- 禁止：改变控制流的宏（宏里 `return`）、依赖调用方局部魔法变量名的宏、把宏当左值、在语句表达式里用易冲突的 `ret` 之类名字。
- 不要重复发明已有工具：数组长度用项目内的 `ARRAY_SIZE(x)`（`sizeof(x) / sizeof((x)[0])`，仅用于真正的数组）；结构体成员大小用 `sizeof(((type *)0)->member)` 或 C11 `offsetof` 组合，不要手写一遍容易算错的公式。`min` / `max` 注意类型是否一致。
- `.c` 里尽量不要铺 `#ifdef`。能整函数裁掉就不要裁半个表达式。可能未使用的函数 / 变量标 `__attribute__((unused))`（或项目封装宏），不要为了消 warning 再包一层 `#ifdef`。
- 非平凡 `#if` / `#ifdef` 的 `#endif` 后写注释标明条件：`#endif /* SCHED_LAB_DEBUG */`。

### 3.10 内存分配

- 按指针类型分配：`p = malloc(sizeof(*p));`、`p = calloc(n, sizeof(*p));`，不要写 `sizeof(struct foo)`（改类型时容易漏改）。
- **不要转换 `malloc` 的返回值**（`void *` 可隐式转换成任意对象指针）。
- 每次分配都检查 `NULL`；失败返回 `-ENOMEM` 或 `NULL`，并走清理路径。
- `realloc` 先用临时指针接返回值，失败时保留原块。
- 谁分配谁释放，释放后不要再解引用；成对出现 `malloc`/`free`、`open`/`close`、`pipe`/`close`。
- 禁止 VLA（变长数组）和 `alloca`；`n`、采样条数来自用户输入时必须在堆上分配并检查溢出（`n * sizeof(*p)` 可能溢出）。
- 需要零填充用 `calloc` 或随后 `memset`，不要假设 `malloc` 返回已清零内存。

### 3.11 返回值与 bool

动作型返回错误码，谓词型返回成功与否，不要混用：

```c
/* 命令：0 成功，负值失败 */
int sched_lab_run_child(int id, int64_t total_ms, int64_t resol_ms);

/* 谓词：true / false */
bool sched_lab_args_ok(const struct sched_lab_args *args);
```

- 计算型函数（返回真正结果而非成败）不受此规则约束；指针失败用 `NULL`。
- 对外函数必须遵守；`static` 函数也建议遵守。
- `bool` 用 `true` / `false`，不要用 `!!` 把整数挤成 0/1 再塞进 `bool`（隐式转换已足够）。
- 结构体若在意布局 / 大小，不要用 `bool` 成员（其对齐随 ABI 变化）；多个开关考虑 bitfield 或 `uint8_t` flags。函数若有一堆 true/false 参数，可收成 flags。

### 3.12 数据结构与并发

用户态同样没有垃圾回收。本实验的并发单元是**进程**。若对象会离开单进程创建/销毁的范围（被放进管道对端、被父进程在 `wait` 之后读取），必须有明确所有权；不要假设 `fork` 之后两边可以同时改同一块堆内存而不出问题——那是写时复制下的独立副本，不是共享。

需要跨进程共享时（管道、共享内存），审查时能回答：谁分配、谁持有、谁释放、失败路径有没有漏、子进程是否错误地 `free` 了只属于父进程的表。

### 3.13 语言特性取舍

| 特性 | 本项目态度 |
| --- | --- |
| GNU 语句表达式、嵌套函数 | 禁止（嵌套函数）；语句表达式仅在无 `static inline` 替代且极局部时才考虑 |
| VLA / `alloca` | 禁止 |
| `goto` | 仅用于集中清理，不用于构造复杂控制网 |
| 位域 | 可对 flags 使用；注意实现定义的布局，不要当可移植 ABI |
| `inline` | 短函数或替代宏；见 3.6 |
| 内联汇编 | 默认禁止；C 能写的不要上 asm |
| 编译器扩展 | 仅 GCC/Clang 上已广泛使用且本规范点名的属性（`__attribute__`） |
| C23 / 编译器私有扩展 | 禁止 |
| 空指针 | 用 `NULL`，不用 `0` |
| `const` | 不修改的指针参数与局部能 `const` 就 `const` |
| 自增 | 单独一句时 `++i` / `i++` 均可；不要把副作用塞进复杂表达式 |

其它常用约定：

- 整数：毫秒、进度、进程个数等与测量相关的字段用定宽类型（优先 `int64_t` 表示毫秒）；循环下标等局部可用 `int` / `size_t`。注意有符号与无符号混用。
- `sizeof` 优先对表达式：`sizeof(*p)`、`sizeof(buf)`。
- `switch` 覆盖所有枚举值或有 `default`；贯穿必须显式 `fallthrough;`（C23 前用注释 `/* fallthrough */` 或编译器认可的 `fallthrough` 宏）。
- 不要在源文件里留下 trailing whitespace。

## 4. 项目专项：进程调度观测

本实验用多进程忙等消耗 CPU，并按固定 CPU 时间间隔采样。除内核风格外，审查必须检查：

1. **进程模型是进程，不是线程**  
   用 `fork`（或明确文档化且语义等价的方式）创建 `n` 个子进程。父进程 `waitpid`（或等价）回收**全部**子进程后再退出。检查 `fork` / `wait` 返回值；失败路径上已创建的子进程必须收尸，禁止留下僵尸进程。子进程记录里的 ID 是 `0` ~ `n-1` 的实验编号，不要和 OS `pid` 混用而不加说明。不要在未说明的情况下把 `n` 实现成 `n` 条 `pthread`。

2. **CPU 时间与墙钟时间分开**  
   - 退出条件：该进程消耗了 `total` 毫秒 **CPU 时间**（`clock_gettime(CLOCK_PROCESS_CPUTIME_ID)` 或等价，如 `getrusage`）。  
   - 输出的「经过的时间」：从**整个程序开始**到该采样点的墙钟时间（优先 `CLOCK_MONOTONIC`），单位毫秒。  
   - 采样间隔 `resol` 对齐的是 CPU 时间进度（每再消耗 `resol` 毫秒 CPU 记一条），不是 `sleep(resol)`。  
   - 忙等循环中禁止插入会让出 CPU 的 `sleep` / `nanosleep`；本实验要的就是调度器眼里的可运行负载。

3. **采样缓冲与输出互斥**  
   每条记录三个字段：`id`、`elapsed_ms`、`progress`（%）。全部进程结束后用制表符分隔、逐行输出。子进程运行期间不要直接抢 `stdout`（多进程同时 `printf` 会交错，无法分析）。应由父进程汇总打印，或子进程结束后按约定有序写出（管道、临时缓冲等）。谁分配缓冲、谁在 `wait` 之后释放，签名上要看得出来。

4. **参数、溢出与进度**  
   校验 `n > 0`、`total > 0`、`resol > 0`，并处理 `resol > total`、不能整除等情况（策略写进注释或用法说明，不要静默给出空输出）。毫秒换算用 64 位，`timespec` 的 `tv_sec` / `tv_nsec` 转毫秒要防溢出。`n * sizeof(*sample)` 一类乘法先检查溢出再 `calloc`。进度用已消耗 CPU / `total`，最后一次采样应达到或明确逼近 100%。

5. **资源在所有路径上释放**  
   管道 fd、动态数组、PID 表，错误返回路径也要 `close` / `free` / `waitpid`。用 goto 集中清理，禁止只在 happy path 里释放。`fork` 之后注意：子进程应关掉只属于父进程的 fd，父进程不要 `free` 仍被孩子使用的共享映射（若使用）。

6. **运行条件写清楚**  
   逻辑 CPU 个数会改变观测图像；绑核（如 `taskset`）是实验变量，不是隐式依赖。用法写在 stderr 帮助或 `docs/`：三个参数含义、输出三列格式、CPU 时间而非墙钟。不要把「必须看见完美 round-robin」写进 `assert`——程序负责出数据，调度结论放在实验分析里。

7. **与 C++ 代码共存时**  
   C 头保持 C 可编译；C++ 侧通过 `extern "C"` 调用。不要在 `.c` 里写 C++，也不要在 `.cpp` 里按本文的 Tab-8 去「统一」C++ 文件。

## 5. 代码审查清单

审查人按顺序看。任何一项不通过，应在评论中指出并要求修改（或明确豁免原因）。

### 设计

- [ ] 改动范围单一，没有顺手大重构
- [ ] 接口能看懂所有权、生命周期和失败语义（动作型 vs 谓词型 vs 指针）
- [ ] 没有引入 C23、VLA、嵌套函数、非标准扩展
- [ ] `n` 个**进程**同时跑；父进程等待全部子进程；无僵尸进程
- [ ] 忙等消耗的是 CPU 时间；采样不靠 `sleep`

### 正确性

- [ ] `fork` / `waitpid` / `clock_gettime` 等返回值已检查
- [ ] 无悬空指针、重复释放、fd 泄漏；goto 清理标签顺序正确；`fork` 失败会收尸
- [ ] 墙钟（elapsed）与 CPU 时间（total / resol / progress）未混用
- [ ] 毫秒与 `n * sizeof` 用足够宽度，无有符号混用导致的环绕
- [ ] `malloc`/`calloc` 已检查；`sizeof(*p)`；未转换 `void *`
- [ ] 输出在全部结束后打印，制表符三列，ID 为 `0` ~ `n-1`
- [ ] 没有依赖未定义行为（含错误的别名、越界、未初始化）

### 风格

- [ ] 命名符合第 3.4 节；结构体未随意 typedef
- [ ] 头文件自包含，include 顺序正确；对外 C 头有 `extern "C"`
- [ ] 已 `clang-format`（Linux / Tab 8 / 80 列）；源文件为 `.c`
- [ ] 花括号：函数开括号另起一行；语句块 K&R；`*` 贴名字
- [ ] 文件内符号 `static`；跨文件符号有 `sched_lab_` 前缀
- [ ] `NULL` / `const` / `bool` 使用正确；无行尾空白

### 可维护性

- [ ] 公开 API 有必要注释（尤其是时钟选择、采样与输出时机）
- [ ] 参数解析、忙等采样、回收打印没有复制粘贴成多份微差逻辑
- [ ] 函数长度与局部变量数量仍在可审范围内
- [ ] 日志足够定位 `fork` 失败、参数非法、时钟失败
- [ ] 实验限制（未绑核、`n` 上限、输出缓冲策略）已写明

## 6. 豁免

允许偏离本文或内核编码风格的情况：

1. **第 1.1 节已列出的用户态映射**（类型、分配器、日志、错误码等）。
2. **对接 POSIX C API**：必须使用的标识符、宏和调用约定（`fork`、`waitpid`、`clock_gettime` 等）。
3. **第三方头文件要求的 include 形式或宏。**
4. **clang-format 无法合理折行的超长字面量、帮助文本；以及按规范不得拆开的日志字符串。**

豁免须在代码旁用一两句注释说明原因。没有注释的偏离，审查默认不通过。

## 7. 参考

- Linux Kernel Coding Style: [https://docs.kernel.org/process/coding-style.html](https://docs.kernel.org/process/coding-style.html)
- 内核树原文：`Documentation/process/coding-style.rst`
- clang-format：Linux / K&R，`IndentWidth` / `TabWidth` 为 8，`UseTab: Always`，`ColumnLimit: 80`
- 内核 clang-format 说明：`Documentation/dev-tools/clang-format.rst`
- The C Programming Language, 2nd ed. (K&R)
- POSIX：`fork(2)`、`waitpid(2)`、`clock_gettime(2)`（`CLOCK_MONOTONIC`、`CLOCK_PROCESS_CPUTIME_ID`）
- 本项目 C++ 规范：`code_review_cpp.md`
