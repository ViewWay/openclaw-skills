# C 标准库 API 速查 (C Standard Library Quick Reference)

> 来源：cppreference/SKILL.md 的 C 标准库部分
> 覆盖范围：<string.h> <stdlib.h> <stdio.h> <math.h> <ctype.h> <time.h> <signal.h> <assert.h> <errno.h> <stdint.h> <stddef.h> <stdbool.h> <inttypes.h> <limits.h> <float.h>
> 完整版本见：~/.openclaw/workspace/skills/cppreference/SKILL.md

---

## <string.h> / <cstring> — 字符串操作

### memcpy / memmove / memset

```c
void *memcpy(void *dest, const void *src, size_t n);   // 不能处理重叠
void *memmove(void *dest, const void *src, size_t n);  // 可以处理重叠
void *memset(void *s, int c, size_t n);                // 填充
int memcmp(const void *s1, const void *s2, size_t n);  // 比较
```

### strcpy / strncpy / strlcpy

```c
char *strcpy(char *dest, const char *src);                          // 不检查长度！
char *strncpy(char *dest, const char *src, size_t n);                // 不保证 \0 终止
size_t strlcpy(char *dest, const char *src, size_t size);           // BSD/macOS 推荐
```

**推荐**：`snprintf(dst, sizeof(dst), "%s", src)` 最安全

### strcat / strncat / strlcat

```c
char *strcat(char *dest, const char *src);   // 不检查长度！
char *strncat(char *dest, const char *src, size_t n);
```

### strlen / strcmp / strncmp

```c
size_t strlen(const char *s);                          // O(n)
int strcmp(const char *s1, const char *s2);            // 返回值不要假设只有 -1,0,1
int strncmp(const char *s1, const char *s2, size_t n);
```

### strchr / strrchr / strstr / strtok

```c
char *strchr(const char *s, int c);        // 正向查找
char *strrchr(const char *s, int c);       // 反向查找
char *strstr(const char *haystack, const char *needle);  // 子串查找
char *strtok(char *str, const char *delim);             // 分割（非线程安全）
char *strtok_r(char *str, const char *delim, char **saveptr);  // 线程安全
```

### strerror

```c
char *strerror(int errnum);  // errno 转字符串
```

---

## <stdlib.h> / <cstdlib> — 通用工具

### 内存管理

```c
void *malloc(size_t size);              // 分配未初始化内存
void *calloc(size_t nmemb, size_t size); // 分配并清零
void *realloc(void *ptr, size_t size);   // 调整大小
void free(void *ptr);                    // 释放
```

### 字符串转数字

```c
int atoi(const char *nptr);                // 无错误检查！
long strtol(const char *nptr, char **endptr, int base);  // 推荐
long long strtoll(const char *nptr, char **endptr, int base);
unsigned long strtoul(const char *nptr, char **endptr, int base);
double strtod(const char *nptr, char **endptr);
```

### 排序与搜索

```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
void *bsearch(const void *key, const void *base,
              size_t nmemb, size_t size,
              int (*compar)(const void *, const void *));
```

### 随机数

```c
int rand(void);        // 伪随机，不安全
void srand(unsigned int seed);
// 安全随机数：arc4random() / arc4random_uniform() (BSD/macOS)
```

### 程序控制

```c
void exit(int status);         // 正常退出
void abort(void);              // 异常终止
int atexit(void (*func)(void)); // 退出时调用
int system(const char *command);
char *getenv(const char *name);
```

### 数学辅助

```c
int abs(int j);
long labs(long j);
long long llabs(long long j);
div_t div(int numer, int denom);
```

---

## <stdio.h> / <cstdio> — 输入输出

### 格式化输出

```c
int printf(const char *format, ...);
int fprintf(FILE *stream, const char *format, ...);
int sprintf(char *str, const char *format, ...);       // 不检查长度！
int snprintf(char *str, size_t size, const char *format, ...);  // 推荐
```

### 格式化输入

```c
int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
```

### 文件操作

```c
FILE *fopen(const char *pathname, const char *mode);
int fclose(FILE *stream);
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
int fseek(FILE *stream, long offset, int whence);
long ftell(FILE *stream);
void rewind(FILE *stream);
int fflush(FILE *stream);
```

### 行/字符操作

```c
char *fgets(char *s, int size, FILE *stream);   // 推荐（有长度限制）
char *gets(char *s);                             // 已移除！永远不要用
int fputs(const char *s, FILE *stream);
int fgetc(FILE *stream);
int fputc(int c, FILE *stream);
int ungetc(int c, FILE *stream);
int feof(FILE *stream);
int ferror(FILE *stream);
```

### 文件管理

```c
int remove(const char *pathname);
int rename(const char *oldpath, const char *newpath);
FILE *popen(const char *command, const char *type);  // 管道
int pclose(FILE *stream);
```

---

## <math.h> / <cmath> — 数学

```c
double fabs(double x);           // 绝对值
double fmod(double x, double y); // 取模
double floor(double x);          // 下取整
double ceil(double x);           // 上取整
double round(double x);          // 四舍五入
double trunc(double x);          // 截断
double sqrt(double x);           // 平方根
double pow(double x, double y);  // 幂
double exp(double x);            // e^x
double log(double x);            // ln(x)
double log2(double x);
double log10(double x);
double sin(double x); double cos(double x); double tan(double x);
double asin(double x); double acos(double x); double atan2(double y, double x);
int isnan(double x); int isinf(double x); int isfinite(double x);
double copysign(double x, double y);
```

---

## <ctype.h> / <cctype> — 字符分类

```c
int isalnum(int c);  int isalpha(int c);  int isdigit(int c);
int islower(int c);  int isupper(int c);  int isspace(int c);
int tolower(int c);  int toupper(int c);
```

---

## <time.h> / <ctime> — 时间

```c
time_t time(time_t *tloc);
struct tm *localtime(const time_t *timep);
struct tm *gmtime(const time_t *timep);
char *ctime(const time_t *timep);
char *asctime(const struct tm *tm);
size_t strftime(char *s, size_t max, const char *format, const struct tm *tm);
clock_t clock(void);  // CPU 时间
```

---

## <signal.h> / <csignal> — 信号

```c
void (*signal(int signum, void (*handler)(int)))(int);
int raise(int sig);
// SIGINT SIGTERM SIGKILL SIGSEGV SIGBUS SIGFPE SIGPIPE
```

---

## <assert.h> / <cassert> — 断言

```c
#include <assert.h>
assert(expression);  // 条件为假时 abort
// NDEBUG 宏定义后 assert 不生效
```

---

## <errno.h> / <cerrno> — 错误码

```c
extern int errno;
// 常见值：ENOENT EACCES EAGAIN EINTR ENOMEM EINVAL
#include <string.h>
strerror(errno);  // 转字符串
perror("func");   // 打印到 stderr
```

---

## <stdint.h> / <cstdint> — 定宽整数

```c
int8_t   int16_t   int32_t   int64_t
uint8_t  uint16_t  uint32_t  uint64_t
intptr_t  uintptr_t
int_fast8_t  int_least8_t  intmax_t  uintmax_t
SIZE_MAX  INT64_MAX  INT64_MIN  UINT64_MAX
```

---

## <stdbool.h> / <cstdbool> — 布尔

```c
#define bool _Bool
#define true 1
#define false 0
#define BOOL_MAX 1
// C23 后 bool/true/false 是内置关键字
```

---

## <inttypes.h> / <cinttypes> — 格式化宏

```c
#include <inttypes.h>
PRId64   // "%ld" 或 "%lld"（跨平台 int64_t 格式）
PRIu64   // 无符号
PRIx64   // 十六进制
SCNd64   // scanf 格式
```

---

## <limits.h> / <climits> — 极限值

```c
INT_MIN  INT_MAX  UINT_MAX  LONG_MAX  LLONG_MAX
CHAR_BIT  // 通常为 8
SHRT_MIN  SHRT_MAX  USHRT_MAX
```

---

## <float.h> / <cfloat> — 浮点极限

```c
FLT_EPSILON  DBL_EPSILON  LDBL_EPSILON
FLT_MAX  DBL_MAX  LDBL_MAX
FLT_MIN  DBL_MIN  LDBL_MIN
FLT_DIG  DBL_DIG  LDBL_DIG  // 有效数字位数
```

---

> 📖 完整 API 文档详见 cppreference/SKILL.md，包含每个函数的详细参数、返回值、复杂度和安全说明。
