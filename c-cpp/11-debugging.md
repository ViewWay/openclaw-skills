# 调试与性能分析 (Debugging & Performance Analysis)

> 来源：cppreference/SKILL.md 的调试部分 + compiler-toolchain/SKILL.md 的性能分析部分 + c-cpp-tutorial/SKILL.md 的调试部分
> 覆盖范围：Sanitizers、gdb、Valgrind、perf、VTune、火焰图、Core Dump、静态分析

---

## 1. Sanitizers

### 地址检查（ASan）

```bash
# 编译
gcc -g -fsanitize=address prog.c
clang -g -fsanitize=address prog.cpp

# 运行
ASAN_OPTIONS=detect_leaks=1 ./prog
```

**检测**：内存越界、use-after-free、double-free、内存泄漏、栈溢出

### 未定义行为检查（UBSan）

```bash
gcc -g -fsanitize=undefined prog.c
```

**检测**：整数溢出、空指针解引用、未初始化读取、移位越界

### 线程竞争检查（TSan）

```bash
gcc -g -fsanitize=thread prog.cpp
```

**检测**：数据竞争、死锁

### 内存检查（MSan）

```bash
clang -g -fsanitize=memory prog.cpp
```

**检测**：未初始化内存读取

### 组合使用

```bash
gcc -g -fsanitize=address,undefined prog.c
# 注意：TSan 不能与 ASan 同时使用
```

### 注意事项
- Sanitizer 仅用于**测试/开发**，生产构建禁用
- ASan 开销约 2x，UBSan 约 1.5x，TSan 约 5-15x
- 开启 `-g` 保留调试信息以获得可读的堆栈

---

## 2. gdb 常用命令

### 启动与基本操作

```bash
gdb ./prog
gdb -tui ./prog          # TUI 模式
gdb ./prog core          # 分析 core dump
```

### 命令速查

| 命令 | 缩写 | 说明 |
|------|------|------|
| `break main` | `b` | 设置断点 |
| `break main.c:42` | | 在文件行设置断点 |
| `break func if x > 0` | | 条件断点 |
| `run` | `r` | 运行程序 |
| `run arg1 arg2` | | 带参数运行 |
| `next` | `n` | 下一行（不进入函数） |
| `step` | `s` | 下一行（进入函数） |
| `finish` | `fin` | 运行到当前函数返回 |
| `continue` | `c` | 继续运行 |
| `print x` | `p` | 打印变量 |
| `print *arr@10` | | 打印数组前10个元素 |
| `print &var` | | 打印地址 |
| `print/x var` | | 十六进制打印 |
| `display var` | | 每次停止时自动打印 |
| `watch var` | | 设置观察点（变量变化时停止） |
| `backtrace` | `bt` | 打印调用栈 |
| `frame N` | `f` | 切换栈帧 |
| `info locals` | | 局部变量 |
| `info args` | | 函数参数 |
| `info threads` | | 列出线程 |
| `thread N` | | 切换线程 |
| `set var x = 42` | | 修改变量值 |
| `list` | `l` | 查看源码 |
| `quit` | `q` | 退出 |

### 多线程调试

```bash
(gdb) info threads          # 列出所有线程
(gdb) thread 2              # 切换到线程 2
(gdb) thread apply all bt   # 所有线程的调用栈
```

### 条件断点

```bash
(gdb) break main.c:100 if i == 42
(gdb) break func if strcmp(name, "test") == 0
```

---

## 3. Valgrind

### memcheck（内存错误检测）

```bash
valgrind --leak-check=full --show-leak-kinds=all \
    --track-origins=yes --verbose ./prog

# 仅检查泄漏
valgrind --leak-check=full ./prog
```

### massif（堆内存使用趋势）

```bash
valgrind --tool=massif ./prog
ms_print massif.out.12345
```

### callgrind（缓存/分支模拟分析）

```bash
valgrind --tool=callgrind ./prog
kcachegrind callgrind.out.12345  # 可视化
```

### 注意
- Valgrind 不支持 SIMD 指令（AVX/SSE）
- 开销大（10-50x 慢），开发阶段用 ASan 更快
- 发布前用 Valgrind 做深度检查

---

## 4. CPU 性能分析

### perf（Linux）

```bash
# 采样分析
perf record -g --call-graph dwarf -F 99 ./app

# 文本报告
perf report --stdio

# 全局统计
perf stat ./app

# 查看热点函数
perf report -g --stdio | head -30
```

### 火焰图

```bash
git clone https://github.com/brendangregg/FlameGraph
perf record -g -F 99 ./app
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
# 在浏览器中打开 flame.svg
```

### VTune（Intel）

```bash
vtune -collect hotspots ./app
vtune -report summary
```

### gprof

```bash
gcc -pg -O2 -o app source.c
./app
gprof app gmon.out > profile.txt
```

### 最佳实践
- **perf + FlameGraph** 是 Linux 下 CPU profiling 首选
- 采样频率 99Hz（避免与系统定时器同步）
- `--call-graph dwarf` 比 `fp` 更准确
- VTune 提供最深入的微架构分析

---

## 5. Core Dump 分析

```bash
ulimit -c unlimited          # 启用 core dump
./prog                       # 崩溃后生成 core 文件
gdb ./prog core              # 分析
(gdb) bt                     # 查看崩溃位置
(gdb) info frame             # 当前帧信息
(gdb) print var              # 查看变量
```

### Core Dump 配置

```bash
# 查看存储位置
sysctl kernel.core_pattern

# 自定义存储
sysctl -w kernel.core_pattern=/tmp/core.%e.%p
```

---

## 6. 静态分析

### clang-tidy

```bash
clang-tidy src.cpp -- -std=c++20
clang-tidy src.cpp --checks='modernize-*,bugprone-*,performance-*,readability-*'
```

### cppcheck

```bash
cppcheck --enable=all .
cppcheck --enable=all --suppress=missingInclude .
```

### GCC 静态分析

```bash
gcc -fanalyzer -O2 -c source.c  # GCC 10+
```

---

## 7. 代码覆盖率

### gcov (GCC)

```bash
gcc -fprofile-arcs -ftest-coverage -O0 prog.c
./prog
gcov prog.c
# 生成 prog.c.gcov，查看覆盖率
```

### lcov（gcov 前端）

```bash
lcov --capture --directory . --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_html
```

---

## 8. 二进制分析

```bash
size -A app                          # 段大小
nm -C --size-sort app | tail -20     # 最大符号
objdump -d app | less                # 反汇编
readelf -a app                       # ELF 信息
ldd app                              # 共享库依赖
ldd -r app                           # 未解析符号
strip --strip-unneeded app           # 减体积
```

---

## 9. macOS 工具

```bash
# Instruments
instruments -t "Time Profiler" ./app
instruments -t "Allocations" ./app

# lldb
lldb ./app
(lldb) b main
(lldb) run
(lldb) p variable
(lldb) bt

# heaptrack
heaptrack ./app
heaptrack_print heaptrack.app.*.gz
```

---

## 10. 调试构建推荐配置

```bash
# 调试构建
gcc -std=c17 -Og -g3 -gdwarf-5 -fno-omit-frame-pointer \
    -Wall -Wextra -Wshadow \
    -fsanitize=address,undefined -fno-omit-frame-pointer \
    -o debug_build source.c

# CMake Debug 模式
cmake -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_C_FLAGS="-fsanitize=address,undefined" \
      -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined" ..
```
