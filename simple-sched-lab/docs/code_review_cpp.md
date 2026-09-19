# simple-sched-lab 开发规范与代码审查指南

本文件是 **simple-sched-lab**（进程调度观测实验）的 C++ 开发约定与代码审查依据。
实验目标、命令行与观测步骤以 `experiment.md` 为准。本文只约束 C++ 代码怎么写。

命令行：`./sched_cpp nproc total resol`（含义见 `experiment.md` 第 2.1 节）。C 实现以 `code_review_c.md` 为准。同一改动里两种语言并存时，按文件扩展名分别套用对应规范。

## 1. 规范来源与优先级

1. **Google C++ Style Guide** 为代码风格的权威来源：
   [https://google.github.io/styleguide/cppguide.html](https://google.github.io/styleguide/cppguide.html)
2. 本文是该规范在本项目中的摘录与审查清单，并补充本实验的开发约定。
3. 冲突处理：
   - 风格细节以 Google C++ 规范为准，**下列三项以本项目例外为准**。
   - 构建、目录、进程/时钟/采样输出等项目约定以本文「项目约定」与「项目专项」章节为准。
   - 审查时发现规范未覆盖的问题，以可读性、正确性和一致性为准，并回写本文。

### 1.1 相对 Google 规范的项目例外

| 项 | Google 规范 | 本项目 |
| --- | --- | --- |
| 语言版本 | 当前目标 C++20 | **C++17**，禁止 C++20 / C++23 及编译器私有扩展 |
| 缩进 | 2 空格 | **4 空格** |
| 源文件扩展名 | `.cc` | **`.cpp`** |

除此以外沿用 Google 规范。Google 规范其它条款（命名、头文件、所有权、禁止异常等）仍然适用。

## 2. 开发约定

### 2.1 语言与工具链

| 项 | 约定 |
| --- | --- |
| 语言 | C++17（`CMAKE_CXX_STANDARD=17`），禁止 C++20 / C++23 及编译器私有扩展 |
| 编译器 | GCC 或 Clang，以 Linux GNU 工具链为主 |
| 构建 | CMake ≥ 3.16，使用 `CMakePresets.json` 中的预设 |
| 编译数据库 | `compile_commands.json`（供 clangd 使用） |
| 格式化 | `clang-format`，`BasedOnStyle: Google`，并覆盖 `IndentWidth: 4` |
| 风格检查 | `cpplint` 可作为辅助；审查不以工具通过为唯一标准 |

CMake 用选项隔离两套代码：`SIMPLE_SCHED_BUILD_C` / `SIMPLE_SCHED_BUILD_CPP`。预设：

| 预设 | 作用 |
| --- | --- |
| `simple-sched-lab` | 配置两侧（默认都编） |
| `sched-c` | 只配置 C |
| `sched-cpp` | 只配置 C++17 |

```bash
cmake --preset sched-cpp
cmake --build --preset sched-cpp
```

本地提交前至少完成：

1. 代码可编译。打开 `-Wall -Wextra`；warning 要看、要处理，但**不要** `-Werror`（警告不导致构建失败）。
2. `clang-format` 已格式化本次改动。
3. 手动走通本次改动对应的实验路径（至少：`nproc=1` 与 `nproc>1` 各跑一遍，检查输出列与进度）。

### 2.2 目录与文件

两套代码分目录隔离，不要把 C 和 C++ 源文件放进同一个构建目标：

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

随着实现展开，在**各自目录内**按职责拆分，而不是把 fork、计时、采样缓冲、输出全堆进一个 `main`，也不要跨目录互相 `#include`。C++ 头仍放在 `sched_cpp/` 下（例如 `sample_log.h`），不要做成仓库根上的共用 `include/`。

文件命名：

- 源文件 `.cpp`，头文件 `.h`。本项目统一使用**小写 + 下划线**：`busy_loop.cpp`、`sample_log.h`、`time_util.cpp`。
- 测试文件：`foo_test.cpp`。
- 头文件与实现成对出现；仅 `main.cpp`、测试文件可以没有对应头文件。
- 不要使用已在系统路径中常见的名字（如 `time.h`、`sched.h`），避免与系统头冲突。

头文件保护宏按仓库路径生成，例如 `sched_cpp/sample_log.h`：

```cpp
#ifndef SCHED_LAB_SAMPLE_LOG_H_
#define SCHED_LAB_SAMPLE_LOG_H_
// ...
#endif  // SCHED_LAB_SAMPLE_LOG_H_
```

### 2.3 命名空间

业务代码放在 `sched_lab` 命名空间中。禁止：

- `using namespace xxx;`（含 `using namespace std;`）
- 匿名以外的 inline namespace
- 在头文件中引入会污染调用方的 using 声明（个别别名除外，见 Google 规范）

实现文件可用内部链接（unnamed namespace 或 `static`）隐藏仅本文件使用的符号。

### 2.4 Git 与审查节奏

项目目前可以没有远程仓库，但本地改动仍按可审查单元组织：

- **一次提交只做一件事**：功能、重构、格式化不要混在同一提交。
- 提交说明写清「为什么」，而不是罗列改了哪些文件。
- 审查以 diff 为单位：先看设计与接口，再看实现与风格。
- 实验性简化（例如未处理 `nproc` 过大、未绑定逻辑 CPU）可以先合入，但必须在注释或 `docs/` 中标明限制。

### 2.5 错误处理策略

本项目遵循 Google 约定：**不使用 C++ 异常**作为控制流。

- `fork`、`waitpid`、`clock_gettime`、管道/`write` 等失败：返回错误码或项目统一的 `Status` / `bool` + 日志，并保证已经获取的资源被释放、已经创建的子进程被回收。
- 构造失败：用工厂函数（返回 `std::unique_ptr` 或错误码），不要让半成品对象流入后续逻辑。
- 断言（`assert` / `CHECK`）只用于「不可能发生」的内部不变量，不用于可预期的参数错误、`fork` 失败、时钟不可用。
- 不要 `catch (...)` 吞掉错误；本项目默认关闭异常，也不引入 `std::exception_ptr`。
- `main` 里校验 `argc`/`argv` 失败时向 stderr 打印用法并以非零状态退出。

## 3. 代码规范摘要（Google C++）

以下为审查时必须核对的要点。细节以官方指南为准。

### 3.1 头文件

- 头文件必须自包含：能单独编译，包含自己用到的所有头。
- 用到的符号在本文件直接 `#include`，不要依赖间接包含。
- 尽量避免前向声明；尤其不要前向声明本项目不拥有的类型（含 `std::`）。
- 头文件中的函数定义应很短（大约 10 行以内）；更长的实现放到 `.cpp`，除非模板/`constexpr` 必须可见。
- Include 顺序（组与组之间空一行，组内字母序）：

  1. 对应头文件（`foo.cpp` 先包含 `foo.h`）
  2. C 系统头（`<unistd.h>` 等）
  3. C++ 标准库头（`<string>` 等）
  4. 第三方库头（本实验通常没有）
  5. 本项目头文件

  系统 / 标准库用 `<>`；本项目头用 `""`，路径相对源码根，禁止 `../`。

```cpp
#include "sample_log.h"

#include <sys/wait.h>
#include <unistd.h>

#include <cstdint>
#include <vector>
```

### 3.2 作用域与资源

- 局部变量在最小作用域内声明，并在声明时初始化。
- 动态资源必须有明确所有者。优先 `std::unique_ptr` 表达独占所有权；共享所有权（`std::shared_ptr`）需要充分理由，且对象最好不可变。
- 禁止 `std::auto_ptr`，禁止无主的 `new`/`delete` 配对散落在业务路径中。
- 全局 / 静态对象的构造与析构顺序不可依赖；有非平凡析构的静态对象要特别谨慎。
- `thread_local` 仅在确实需要线程局部状态时使用，并写明生命周期。本实验默认用**进程**产生并发负载；不要在未说明的情况下把 `nproc` 实现成同样数量的线程。

### 3.3 类

- 构造函数只做初始化，避免复杂失败路径；需要失败则用工厂。
- 禁止隐式转换：单参数构造函数与转换运算符标 `explicit`。
- 需要拷贝/移动时成套定义或删除（遵循 Rule of Five / Zero）。优先 Rule of Zero。
- `struct` 用于纯数据聚合（如一条采样记录）；`class` 用于有不变量或行为的类型。
- 继承用于「是一个」的接口/实现关系，优先组合；避免深层次公开继承。
- 运算符重载克制：只在含义与内置类型一致时使用。
- 数据成员默认 `private`。声明顺序：`public` → `protected` → `private`；同类声明中类型、常量、工厂、构造/析构、其它函数、数据成员分组清晰。

### 3.4 函数

- 函数保持短小、单一职责。过长函数在审查中应被要求拆分（例如：参数解析、子进程忙等与采样、父进程回收与打印，不要揉成一个 `main`）。
- 输入只读用 `const T&` 或轻量类型按值；可选所有权用 `std::unique_ptr<T>`；输出优先返回值，避免非 const 引用输出参数，除非必要。
- 重载必须让调用方一眼能分辨意图，避免仅靠隐式转换区分。
- 默认参数只用于不会造成重载歧义的简单情况。
- 尾随返回类型仅在必要（如复杂 `decltype`）时使用。

### 3.5 语言特性取舍

| 特性 | 本项目态度 |
| --- | --- |
| 异常 | 不使用 |
| RTTI（`dynamic_cast` / `typeid`） | 避免；优先虚函数。测试中可例外 |
| C 风格强制转换 | 禁止（转 `void` 除外）；用 `static_cast` / `const_cast` / `reinterpret_cast` / 花括号初始化 |
| 宏 | 尽量不用；必须时全大写并加 `SCHED_LAB_` 前缀 |
| `using` 别名 | 可以，名字按类型命名规则 |
| `auto` | 类型对读者明显时可使用；接口、以及会掩盖所有权/整数宽度的地方写明类型 |
| Lambda | 短小、局部使用；避免过度捕获与隐式拷贝 |
| 模板元编程 | 克制，不为炫技 |
| C++20 起特性（modules、协程、Concepts、`std::span` 等） | 不使用 |
| 非标准扩展 | 禁止 |
| `std::move` / 右值引用 | 用于移动构造/赋值、完美转发；不要把 `&&` 随意写进普通 API |

其它常用约定：

- 空指针用 `nullptr`，不用 `NULL` 或 `0`。
- 整数：毫秒、进度、进程个数等与测量相关的字段用定宽类型（`int32_t` / `uint32_t` / `int64_t` 等）；循环下标等局部可用 `int` / `size_t`。注意无符号陷阱与毫秒乘法溢出（优先 `int64_t`）。
- 能 `const` 就 `const`；编译期常量用 `constexpr`。
- 自增自减优先前置：`++i`。
- `sizeof` 优先对变量或类型对象使用，避免手写依赖类型名且易过期的形式。
- `switch` 必须覆盖所有枚举值或有 `default`；`case` 贯穿必须显式注释 `[[fallthrough]]`。

### 3.6 命名

目的：看到名字就能判断它是类型、函数、变量还是常量。

| 类别 | 规则 | 示例 |
| --- | --- | --- |
| 文件 | 小写下划线 + `.cpp` / `.h` | `sample_log.cpp` |
| 类型 | PascalCase | `SampleRecord`, `ChildWorker` |
| 函数 | PascalCase | `ParseArgs()`, `RunBusyLoop()` |
| 访问器 / 设置器 | 可 snake_case | `id()`, `set_progress()` |
| 变量 / 参数 | snake_case | `child_id`, `elapsed_ms` |
| class 数据成员 | snake_case + 尾下划线 | `total_ms_`, `resol_ms_` |
| struct 数据成员 | snake_case，无尾下划线 | `id`, `elapsed_ms`, `progress` |
| 编译期/程序期常量 | `k` + PascalCase | `kMaxChildren` |
| 枚举值 | 同常量 | `enum class Status { kOk, kForkFailed }` |
| 命名空间 | snake_case | `sched_lab` |
| 宏 | 全大写 + 项目前缀 | `SCHED_LAB_DISALLOW_COPY` |

名字要表达意图。不要靠删字母缩写（禁止 `prcs`、`schd` 这类项目外看不懂的缩写）。调度/进程领域通用词（CPU、PID、fork、wait）可以使用。

包容性用语：注释和标识符避免 master/slave、blacklist/whitelist 等说法，改用 primary/secondary、blocklist/allowlist 等中性词。

### 3.7 注释

- 用 `//`，不把注释块当成代码的替代文档。
- 文件头：说明该文件职责（不必重复版权套话）。
- 类与公开函数：说明用途、所有权、线程/进程约定、失败语义。对「做什么」注释，不要复述代码「怎么做」。
- 时钟选择（为何用进程 CPU 时间而不是经过的时间）、采样为何不能 `sleep`、子进程为何不得直接写 stdout 等**非显然约束**，必须在实现处注释。
- `TODO` 格式：`// TODO(owner): 要做什么及原因`。

注释、标识符和提交说明：本项目文档可用中文；代码标识符使用英文。不要中英混杂命名。

### 3.8 格式

以 `clang-format`（Google 底版 + 4 空格缩进）为准，人工审查关注工具覆盖不到的可读性。放在 `sched_cpp/.clang-format`，只约束 C++ 树：

```yaml
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 80
```

- 每行最多 80 列（include、头保护、无法折行的 URL/字面量除外）。
- 只使用空格，缩进 **4 空格**，禁止 Tab。
- 指针/引用：`int* ptr`、`const std::string& name`（`*` / `&` 贴类型）。
- 花括号：K&R / Google 风格，即使单句 `if` 也建议加大括号，避免后续修改踩坑。
- 命名空间不额外缩进。
- 水平空格：二元运算符两侧有空格；括号内侧无空格。
- 垂直空格：函数之间空一行；不要大段空行。

## 4. 项目专项：进程调度观测

本实验用多进程忙等消耗 CPU，并按固定工作量间隔采样。实验语义以 `experiment.md` 为准。除 Google 规范外，审查必须检查：

1. **用进程，不用线程**  
   用 `fork`（或文档中写明的等价方式）创建 `nproc` 个子进程。父进程 `waitpid`（或等价接口）回收**全部**子进程后再退出。检查 `fork` / `wait` 的返回值；失败路径上已创建的子进程必须回收，禁止留下僵尸进程。记录里的 ID 是 `0` ~ `nproc-1` 的实验编号，不要和操作系统的 `pid` 混用而不加说明。

2. **CPU 时间与经过的时间分开**  
   - `total` / `resol` 是 CPU 工作量（独占处理器时大约要跑多久），不是 `sleep`。  
   - 输出的「经过的时间」：从程序开始到该采样点现实中过了多久（优先 `CLOCK_MONOTONIC`），单位毫秒。  
   - 可以用循环标定，也可以用进程 CPU 时钟等到 `resol` 毫秒；两种做法都要在 `fork` 前准备好计量基准。  
   - 忙等循环中禁止 `sleep` / `nanosleep`。

3. **采样与输出**  
   每条记录三列：`id`、`elapsed_ms`、`progress`（%）。忙等过程中不要打印。子进程结束时自己打印、或父进程汇总后再打印都可以。谁分配缓冲区、谁释放，要能从接口看出来。

4. **参数、溢出与进度**  
   校验 `nproc > 0`、`total > 0`、`resol > 0`，且 `total` 能被 `resol` 整除。毫秒换算用 64 位，`timespec` 转毫秒要防止溢出。每个进程 `nrecord = total / resol` 条，最后一次进度为 100%。

5. **资源在所有路径上释放**  
   管道 fd、动态数组、子进程 PID 表，错误返回路径也要 `close` / `free` / `wait`。禁止只在成功路径里清理。

6. **运行条件写清楚**  
   逻辑 CPU 个数会改变观测结果；绑定到指定逻辑 CPU（如 `taskset`）是实验变量，不要当成默认环境。用法写在 `--help` 或 `docs/`：三个参数含义、输出三列、`total`/`resol` 是 CPU 工作量、`elapsed_ms` 是经过的时间。不要把「必须均匀轮转」写进断言。

7. **与 C 代码共存时**  
   C++ 源遵守本文；C 源遵守 `code_review_c.md`。不要在 `.cpp` 里按内核风格的 Tab、8 列缩进去「统一」。

## 5. 代码审查清单

审查人按顺序看。任何一项不通过，应在评论中指出并要求修改（或明确豁免原因）。

### 设计

- [ ] 改动范围单一，没有顺手大重构
- [ ] 接口能看懂所有权、生命周期和失败语义
- [ ] 没有引入异常、C++20+ 特性、非标准扩展
- [ ] `nproc` 个**进程**同时跑；父进程等待全部子进程；无僵尸进程
- [ ] 忙等消耗的是 CPU 时间；采样不靠 `sleep`

### 正确性

- [ ] `fork` / `waitpid` / `clock_gettime` 等返回值已检查
- [ ] 无悬空指针、重复释放、fd 泄漏；失败路径回收已创建的子进程
- [ ] 经过的时间（`elapsed_ms`）与 CPU 工作量（`total` / `resol` / `progress`）未混用
- [ ] 毫秒计算用足够宽度，无有符号混用导致的环绕
- [ ] 输出为制表符三列，ID 为 `0` ~ `nproc-1`；忙等过程中不打印
- [ ] 没有依赖未定义行为（含错误的 `reinterpret_cast`、类型别名违规）

### 风格

- [ ] 命名符合第 3.6 节
- [ ] 头文件自包含，include 顺序正确
- [ ] 已 `clang-format`（Google + 4 空格）；源文件为 `.cpp`
- [ ] 无 `using namespace`
- [ ] 无 C 风格转换、无无主 `new`/`delete`
- [ ] `const` / `nullptr` / `++i` 使用正确

### 可维护性

- [ ] 公开 API 有必要注释（尤其是时钟选择、采样与输出时机）
- [ ] 参数解析、忙等采样、回收打印没有复制粘贴成多份微差逻辑
- [ ] 日志足够定位 `fork` 失败、参数非法、时钟失败
- [ ] 实验限制（未绑定逻辑 CPU、`nproc` 上限、输出由谁打印）已写明

## 6. 豁免

允许偏离本文或 Google 规范的情况：

1. **第 1.1 节已列出的项目例外**（C++17、4 空格缩进、`.cpp` 扩展名）。
2. **对接 POSIX C API**：必须使用的标识符、宏和调用约定（`fork`、`waitpid`、`clock_gettime` 等）。
3. **第三方头文件要求的 include 形式或宏。**
4. **clang-format 无法合理折行的超长字面量、帮助文本。**

豁免须在代码旁用一两句注释说明原因。没有注释的偏离，审查默认不通过。

## 7. 参考

- Google C++ Style Guide: [https://google.github.io/styleguide/cppguide.html](https://google.github.io/styleguide/cppguide.html)
- clang-format：`BasedOnStyle: Google`，`IndentWidth: 4`
- cpplint: [https://github.com/cpplint/cpplint](https://github.com/cpplint/cpplint)
- POSIX：`fork(2)`、`waitpid(2)`、`clock_gettime(2)`（`CLOCK_MONOTONIC`、`CLOCK_PROCESS_CPUTIME_ID`）
- 本项目 C 规范：`code_review_c.md`
