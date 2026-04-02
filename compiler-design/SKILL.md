# 编译原理

## 词法分析

### 正则表达式定义

**定义**：正则表达式是描述正则语言的代数表达式，由以下规则递归定义：
- 基础符号：a ∈ Σ 匹配单个字符
- 连接：R₁R₂ 匹配 L(R₁)·L(R₂)
- 选择：R₁|R₂ 匹配 L(R₁) ∪ L(R₂)
- Kleene 闭包：R* 匹配 (L(R))*
- 扩展运算符：R+（正向闭包）、R?（可选）、R{n,m}（重复）

**定理（Kleene 定理）**：语言 L 是正则的当且仅当存在有限自动机接受 L。

**证明思路**：
- 正则 → 自动机：对正则表达式进行结构归纳，构造对应的 ε-NFA
- 自动机 → 正则：使用状态消除法或 Arden 引理构造等价正则表达式

**复杂度**：正则表达式转换为 ε-NFA 的时间复杂度为 O(|R|)，其中 |R| 是正则表达式长度。

**应用**：词法分析器生成器（Lex/flex/re2c）、模式匹配、文本处理。

### NFA → DFA 转换（子集构造法）

**定义**：非确定性有限自动机（NFA）的每个状态对同一输入可能有多个转移；确定性有限自动机（DFA）对每个状态和输入有唯一转移。

**定理（子集构造定理）**：对每个 NFA N，存在等价的 DFA D，使得 L(N) = L(D)。DFA 的状态数至多为 2^n，其中 n 是 NFA 的状态数。

**证明思路**：
- DFA 的状态是 NFA 状态的子集
- 初始状态为 ε-closure(q₀)
- 对每个输入符号，计算子集转移：δ'(S,a) = ∪_{q∈S} δ(q,a) 的 ε-closure
- 包含接受状态的子集为接受状态

**算法（子集构造）**：
```
输入：NFA N = (Q, Σ, δ, q₀, F)
输出：等价 DFA D = (Q', Σ, δ', q₀', F')

1. q₀' = ε-closure(q₀)
2. Q' = {q₀'}, 工作列表 = [q₀']
3. while 工作列表非空：
     T = pop(工作列表)
     for each a ∈ Σ:
         U = ε-closure(∪_{q∈T} δ(q,a))
         if U ∉ Q':
             Q' ← Q' ∪ {U}
             push(工作列表, U)
         δ'(T, a) = U
4. F' = {T ∈ Q' | T ∩ F ≠ ∅}
5. 返回 D
```

**复杂度**：O(n²·|Σ|)，其中 n 是 NFA 状态数。最坏情况下生成 2^n 个 DFA 状态。

**应用**：词法分析器实现、正则表达式引擎编译阶段。

### DFA 最小化（Hopcroft 算法）

**定义**：两个状态 p, q 是等价的，如果对任意字符串 w，从 p 和 q 出发都能接受/拒绝 w。最小 DFA 是接受同一语言的状态数最少的 DFA。

**定理（Myhill-Nerode 定理）**：语言 L 的最小 DFA 状态数等于其右同余类的数量。

**证明思路**：通过状态等价关系对 DFA 状态划分，不断细化直到稳定。

**算法（Hopcroft 最小化）**：
```
输入：DFA D = (Q, Σ, δ, q₀, F)
输出：最小 DFA Dmin

1. P = {F, Q\F}  // 初始划分
2. W = {F}  // 工作集（选择较小的划分）
3. while W 非空：
     A = pop(W)
     for each c ∈ Σ:
         X = {q ∈ Q | δ(q,c) ∈ A}
         for each Y ∈ P where X ∩ Y ≠ ∅ and Y\X ≠ ∅:
             用 (X ∩ Y, Y\X) 替换 P 中的 Y
             if Y ∈ W:
                 用 (X ∩ Y, Y\X) 替换 W 中的 Y
             else:
                 将 |X ∩ Y| ≤ |Y\X| 的子集加入 W
4. 合并每个划分中的状态为一个状态
5. 返回最小化 DFA
```

**复杂度**：O(n·log n)，其中 n 是 DFA 状态数。

**应用**：编译器优化、模式匹配加速、硬件电路设计。

### 词法分析器生成器

**定义**：词法分析器生成器根据正则表达式规则自动生成词法分析器代码。

**关键概念**：
- **最长匹配原则**：匹配尽可能长的输入
- **规则优先级**：冲突时选择先定义的规则
- **回溯避免**：预读字符确定最长匹配

**算法**：
```
输入：规则集 R = {(r₁, a₁), ..., (rₙ, aₙ)}，其中 rᵢ 是正则表达式
输出：DFA D 最小化后，带动作标注

1. 为每个 rᵢ 构造 ε-NFA Nᵢ
2. 用新初始状态和 ε 转移合并所有 Nᵢ
3. 子集构造转换为 DFA
4. Hopcroft 算法最小化
5. 对每个接受状态标注优先级最高的动作
6. 生成代码（C/Python/Rust 等）
```

**复杂度**：O(|R| + n²·log n)，其中 |R| 是所有正则表达式长度和。

**应用**：Lex/flex（生成 C）、re2c（生成 C）、PLY/ANTLR（Python）。

---

## 语法分析

### 上下文无关文法（CFG）

**定义**：CFG G = (V, Σ, R, S)，其中：
- V：非终结符集合
- Σ：终结符集合
- R：产生式规则 A → β，其中 A ∈ V，β ∈ (V ∪ Σ)*
- S：起始符号

**定理**：CFG 生成的语言称为上下文无关语言（CFL）。

**关键概念**：
- **推导**：S ⇒* w，应用产生式规则
- **最左推导**：每次替换最左非终结符
- **最右推导**：每次替换最右非终结符
- **左递归**：A ⇒+ Aα（直接/间接）
- **二义性**：存在 w ∈ L(G) 有多棵不同的语法树

**消除左递归**：
```
直接左递归：A → Aα₁ | ... | Aαₙ | β₁ | ... | βₘ
转换为：
A → β₁A' | ... | βₘA'
A' → α₁A' | ... | αₙA' | ε
```

**提取左公因子**：
```
A → αβ₁ | αβ₂ | ... | αβₙ | γ
转换为：
A → αA' | γ
A' → β₁ | ... | βₙ
```

**复杂度**：O(|G|) 消除左递归和提取左公因子。

**应用**：编程语言语法设计、数据格式解析。

### 递归下降分析

**定义**：每个非终结符对应一个递归函数的自顶向下分析方法。

**定理**：给定 CFG G，如果 G 是 LL(1) 的，则可构造递归下降分析器。

**算法**：
```
对每个产生式 A → X₁X₂...Xₖ：
  function A():
    lookahead = peek_token()
    if lookahead ∈ FIRST(X₁X₂...Xₖ):
      for i = 1 to k:
        if Xᵢ 是终结符:
          match(Xᵢ)
        else:
          Xᵢ()  // 递归调用
    else if ε ∈ FIRST(X₁X₂...Xₖ) and lookahead ∈ FOLLOW(A):
      return  // 匹配 ε
    else:
      error()
```

**复杂度**：O(n)，其中 n 是输入长度。每个符号常数时间处理。

**应用**：手工编写解析器（如 C 编译器）、简单 DSL 实现。

### LL(1) 分析

**定义**：LL(1) 文法可从左到右扫描输入，生成最左推导，且只需 1 个符号预读。

**定理（LL(1) 充要条件）**：对同一非终结符 A 的任意两个产生式 A → α | β：
1. FIRST(α) ∩ FIRST(β) = ∅
2. 如果 β ⇒* ε，则 FIRST(α) ∩ FOLLOW(A) = ∅
3. 如果 α ⇒* ε，则 FIRST(β) ∩ FOLLOW(A) = ∅

**FIRST 集合计算**：
```
FIRST(X) = {a ∈ Σ | X ⇒* aγ}
if X ∈ Σ: FIRST(X) = {X}
if X ∈ V:
  for each production X → Y₁Y₂...Yₖ:
    for i = 1 to k:
      FIRST(X) ← FIRST(X) ∪ (FIRST(Yᵢ) \ {ε})
      if ε ∉ FIRST(Yᵢ): break
    if all Yᵢ 可推导 ε: FIRST(X) ← FIRST(X) ∪ {ε}
```

**FOLLOW 集合计算**：
```
FOLLOW(A) = {a ∈ Σ | S ⇒* αAaβ}
1. FOLLOW(S) ← FOLLOW(S) ∪ {$}
2. repeat:
  for each production B → αAβ:
    FIRST(β) \ {ε} ⊆ FOLLOW(A)
    if β ⇒* ε: FOLLOW(B) ⊆ FOLLOW(A)
until no changes
```

**LL(1) 分析表**：
```
M[A, a] = production A → α
if a ∈ FIRST(α): M[A, a] = A → α
if α ⇒* ε and a ∈ FOLLOW(A): M[A, a] = A → α
```

**复杂度**：FIRST/FOLLOW 计算 O(|G|²)，分析 O(n)。

**应用**：LL(1) 解析器生成器（ANTLR4）、表达式求值、配置文件解析。

### LR 分析

**定义**：LR 分析（Left-to-right, Rightmost derivation in reverse）是自底向上、最强大的实用分析方法。

**定理**：LR(1) > LALR(1) > SLR(1) > LR(0)。LR(1) 可处理所有无二义性 CFG。

**LR 分析器组成**：
1. **状态栈**：存储分析状态
2. **符号栈**：存储已识别符号
3. **Action 表**：对 (state, token) 的动作（shift/reduce/goto/accept/error）
4. **Goto 表**：对 (state, nonterminal) 的转移状态

**LR 项（Item）**：A → α·β，其中 · 表示当前位置。
- **LR(0) 项**：仅关注产生式位置
- **LR(1) 项**：(A → α·β, a)，a 是预读符号

**闭包计算**：
```
Closure(I):
  J = I
  repeat:
    for each (A → α·Bβ, a) ∈ J, B → γ ∈ R:
      for each b ∈ FIRST(βa):
        add (B → ·γ, b) to J
  return J
```

**GoTo 转移**：
```
Goto(I, X):
  J = {(A → αX·β, a) | (A → α·Xβ, a) ∈ I}
  return Closure(J)
```

**LR(1) 项集族构造**：
```
C = {Closure({S' → ·S, $})}
repeat:
  for each I ∈ C, X ∈ V ∪ Σ:
    J = Goto(I, X)
    if J ≠ ∅ and J ∉ C:
      C ← C ∪ {J}
until no changes
```

**Action 表填表**：
```
对每个项集 I 和项 (A → α·aβ, b) ∈ I，其中 a ∈ Σ:
  if (A → αa·β, b) ∈ J = Goto(I, a):
    Action[I, a] = shift J

对每个项集 I 和项 (A → α·, a) ∈ I，其中 A ≠ S':
  Action[I, a] = reduce A → α

对每个项集 I 和项 (S' → S·, $) ∈ I:
  Action[I, $] = accept
```

**冲突检测**：
- **shift-reduce 冲突**：同一位置有 shift 和 reduce 动作
- **reduce-reduce 冲突**：同一位置有多个 reduce 动作

**复杂度**：项集族构造 O(|G|·|V ∪ Σ|)，分析 O(n)。

**应用**：Yacc/Bison（LALR(1)）、C/C++/Java 编译器、SQL 解析器。

---

## 语义分析

### 语法树（AST）

**定义**：抽象语法树（Abstract Syntax Tree）是程序语法结构的树状表示，去除语法细节（如括号、分号）。

**定理**：每个语法正确的程序对应唯一的 AST（对无二义文法）。

**AST 结构**：
- 节点类型对应语法类别（表达式、语句、声明等）
- 叶节点：标识符、常量、运算符
- 内部节点：复合结构（if/while/函数调用等）

**遍历方法**：
- **前序遍历**：根 → 左 → 右（深度优先）
- **后序遍历**：左 → 右 → 根（自底向上计算）
- **访问者模式**：分派到不同的处理函数

**算法（构建 AST）**：
```
在语法分析过程中：
  遇到终结符：创建叶子节点
  遇到产生式：递归构建子节点，然后组合

示例：E → E + T
  node = BinaryExpr('+', left=E, right=T)
```

**复杂度**：O(n) 构建，其中 n 是源代码长度。

**应用**：代码分析、转换、生成、优化。

### 符号表

**定义**：符号表记录程序中的标识符及其属性（类型、作用域、存储位置等）。

**定理**：作用域规则保证标识符引用的唯一解析。

**符号表属性**：
- 名称
- 类型（int, float, struct, function 等）
- 作用域（全局/局部/类）
- 存储类（auto/static/extern）
- 偏移量（栈帧中的位置）
- 链接属性（internal/external）

**作用域规则**：
- **词法作用域**：静态确定（C/C++/Java）
- **动态作用域**：运行时确定（Emacs Lisp）

**数据结构**：
- **哈希表**：O(1) 查找，适合单作用域
- **作用域栈**：处理嵌套作用域
- **全局表 + 局部表**：分层存储

**算法（符号表管理）**：
```
enter_scope(): push(作用域栈, new_scope())
exit_scope(): pop(作用域栈)
lookup(name):
  for scope in 作用域栈 (从顶到底):
    if name ∈ scope: return scope[name]
  return error
insert(name, attributes): 当前作用域[name] = attributes
```

**复杂度**：O(depth) 查找，其中 depth 是作用域嵌套深度。

**应用**：类型检查、代码生成、调试信息生成。

### 类型系统

**定义**：类型系统定义表达式的类型及其合法操作。

**定理（类型安全性）**：良类型程序不会发生运行时类型错误。

**类型检查算法**：
```
type_check(expr):
  if expr 是字面量: return 其类型
  if expr 是变量: return 符号表[expr.name].type
  if expr 是二元操作 e₁ op e₂:
    t₁ = type_check(e₁)
    t₂ = type_check(e₂)
    if op 期望类型匹配(t₁, t₂):
      return op 的结果类型
    else:
      type_error()
```

**类型推导（Hindley-Milner）**：
```
算法 W（Damas-Milner）：

1. 为每个表达式变量分配类型变量
2. 为子表达式生成约束方程
3. 统一（unify）类型约束

unify(t₁, t₂):
  if t₁ == t₂: return
  if t₁ 是类型变量: 替换所有 t₁ 为 t₂
  if t₂ 是类型变量: 替换所有 t₂ 为 t₁
  if t₁, t₂ 是函数类型: unify(t₁.param, t₂.param), unify(t₁.return, t₂.return)
  else: error()

例：let f = λx.x
  f : α → α (多态类型)
```

**多态类型**：
- **参数多态**：类型变量（C++ templates, Java generics）
- **特设多态**：重载/子类型（C++, Java）

**复杂度**：Hindley-Milner 推导 O(n·α(n))，其中 α 是反阿克曼函数。

**应用**：函数式语言（Haskell/ML）、类型推断、编译器优化。

### 属性文法

**定义**：属性文法扩展 CFG，为每个符号关联属性值（综合属性/继承属性）。

**定理**：给定依赖无环的属性文法，可构造求值顺序。

**属性类型**：
- **综合属性**（Synthesized）：从子节点计算到父节点（如表达式的值）
- **继承属性**（Inherited）：从父节点/兄弟节点传递到子节点（如声明的作用域）

**依赖图**：
- 节点：每个属性的实例
- 边：属性间的依赖关系
- 有向无环图（DAG）保证可求值

**S-属性文法**（仅综合属性）：
- 可在语法分析过程中自底向上计算
- 示例：表达式的值、AST 构建器

**L-属性文法**（继承属性仅依赖父/左兄弟）：
- 可在递归下降分析中计算
- 示例：符号表维护、类型检查

**算法（属性求值）**：
```
给定依赖图 DAG：
1. 拓扑排序
2. 按顺序计算每个属性
```

**复杂度**：O(n) 构建依赖图，O(n·log n) 拓扑排序。

**应用**：语义分析、代码生成、编译器中间表示构建。

---

## 中间表示

### 三地址码

**定义**：三地址码（Three-Address Code）是每条指令最多包含三个操作数的中间表示。

**形式**：x = y op z，其中 x, y, z 是名字、常量或编译器生成的临时变量。

**指令类型**：
```
1. 二元操作：x = y op z
2. 一元操作：x = op y
3. 赋值：x = y
4. 复制：x = y
5. 无条件跳转：goto L
6. 条件跳转：if x relop y goto L
7. 过程调用：param x, call p, n, return y
8. 索引：x = y[i], x[i] = y
9. 地址：x = &y, x = *y, *x = y
```

**优点**：
- 接近目标机器，易于代码生成
- 独立于源语言，便于优化
- 顺序结构，易于分析

**生成算法**：
```
从 AST 递归生成三地址码：

gen_expr(e):
  if e 是常量 c: return new_temp(c)
  if e 是变量 v: return v
  if e 是 e₁ op e₂:
    t₁ = gen_expr(e₁)
    t₂ = gen_expr(e₂)
    t = new_temp()
    emit(t, '=', t₁, op, t₂)
    return t
```

**复杂度**：O(n) 生成，其中 n 是 AST 节点数。

**应用**：GCC GIMPLE、LLVM IR（SSA 形式）。

### SSA（静态单赋值）

**定义**：SSA（Static Single Assignment）中每个变量只被赋值一次，使用 φ 函数合并控制流。

**定理**：任何程序可转换为 SSA 形式而不改变语义。

**SSA 转换算法**：
```
1. 计算控制流图（CFG）
2. 识别必经节点（Dominance Frontier）
3. 在支配边界插入 φ 函数
4. 重命名变量

支配边界：
  DF(x) = {y | x 支配 z, z 是 y 的前驱, x 不严格支配 y}

φ 函数插入：
  for each variable v:
    for each node x where v 被定义:
      for each y ∈ DF(x):
        在 y 插入 v = φ(v, v, ...)
```

**重命名算法**：
```
Rename(node):
  for each φ 函数 φ(v) in node:
    v_new = new_version(v)
    node.φ[v] = v_new
    push(stack[v], v_new)

  for each statement in node:
    for each use of v in statement:
      statement.v = top(stack[v])
    for each definition v = ... in statement:
      v_new = new_version(v)
      statement.v = v_new
      push(stack[v], v_new)

  for each successor succ in node.succs:
    for each φ function φ(v) in succ:
      succ.φ[v] = top(stack[v])

  for each child in node.dominated_children:
    Rename(child)

  for each v where node 定义 v:
    pop(stack[v])
```

**优点**：
- 使用-定义链明确（use-def chain）
- 数据流分析简化
- 优化机会增多（常量传播、死代码消除）

**复杂度**：O(n·α(n)) 构建和重命名。

**应用**：LLVM IR、GCC GIMPLE（转换后）、JIT 编译器。

### 基本块与控制流图

**定义**：
- **基本块**：只有一个入口和一个出口的连续语句序列
- **控制流图（CFG）**：节点是基本块，边表示可能的控制流

**基本块划分**：
```
识别首指令：
1. 程序第一条指令
2. 条件/无条件跳转的目标
3. 紧跟跳转的指令

每个首指令开始新基本块，直到遇到跳转或下一条首指令
```

**控制流图构造**：
```
节点：每个基本块
边：对每个跳转指令：
  无条件跳转 goto L：从当前块到 L 的块
  条件跳跳转 if x goto L：
    从当前块到 L 的块（条件真）
    从当前块到下一条指令的块（条件假）
```

**必经关系（Dominance）**：
- **必经节点**：节点 d 必经节点 n，如果所有从入口到 n 的路径都经过 d
- **严格必经**：d 必经 n 且 d ≠ n
- **立即必经**：n 的严格必经节点集合中离 n 最近的节点

**必经树算法（Lengauer-Tarjan）**：
```
输入：CFG 的入口节点 n₀
输出：每个节点的立即必经节点 idom[n]

1. DFS 标记节点，计算半必经节点 sdom[n]
2. 求解 idom[n]：

Simplify(n):
  if parent[n] ≠ sdom[n]:
    idom[n] = idom[sdom[n]]
  else:
    idom[n] = sdom[n]

3. 去压缩（可选）
```

**复杂度**：O(n·α(n)) 计算必经关系。

**应用**：SSA 构造、代码优化、数据流分析。

### 数据流分析

**定义**：数据流分析在 CFG 上计算程序属性（如到达定义、活跃变量）。

**数据流分析框架**：
```
给定 CFG 中每个节点 n 的：
- gen[n]：n 生成的属性集合
- kill[n]：n 消除的属性集合
- transfer[n]：传递函数

计算：
  in[n] = ∪_{p∈pred(n)} out[p]
  out[n] = transfer(in[n])
```

**前向分析**（如到达定义）：
```
out[n] = gen[n] ∪ (in[n] \ kill[n])

初始化：
  in[entry] = ∅
  for n ≠ entry: in[n] = ∅  // 向前（可能解）
  或 in[n] = U（全集）       // 向后（必然解）

迭代直到不动点：
  repeat:
    for each n:
      in[n] = ∪_{p∈pred(n)} out[p]
      out[n] = gen[n] ∪ (in[n] \ kill[n])
  until no changes
```

**后向分析**（如活跃变量）：
```
in[n] = use[n] ∪ (out[n] \ def[n])

初始化：
  out[exit] = ∅
  for n ≠ exit: out[n] = ∅

迭代：
  repeat:
    for each n:
      out[n] = ∪_{s∈succ(n)} in[s]
      in[n] = use[n] ∪ (out[n] \ def[n])
  until no changes
```

**常见数据流问题**：
1. **到达定义**（Reaching Definitions）：
   - in[n]：在 n 入口处能到达的定义
   - out[n] = gen[n] ∪ (in[n] \ kill[n])

2. **活跃变量**（Live Variables）：
   - in[n]：在 n 入口处活跃的变量（在后续路径中被使用）
   - in[n] = use[n] ∪ (out[n] \ def[n])

3. **可用表达式**（Available Expressions）：
   - out[n]：在 n 出口处可用的表达式（所有路径上都计算过）
   - out[n] = gen[n] ∩ (in[n] ∩ kill[n]ᶜ)

4. **常量传播**（Constant Propagation）：
   - 追踪变量的常量值
   - 使用 lattice（格）表示可能值（常量/非定值/Top）

**复杂度**：O(N·E)，其中 N 是节点数，E 是边数。使用工作列表算法可优化到 O(N + E)。

**应用**：编译器优化、程序验证、调试信息生成。

---

## 代码优化

### 局部优化

**定义**：在单个基本块内进行的优化，不跨边界。

**1. 常量折叠（Constant Folding）**：
```
x = 3 + 5  →  x = 8
y = x * 2  →  y = 16（如果 x 已知为 8）
```
- 编译时计算常量表达式
- 算法：遍历基本块，识别常量操作数并计算

**2. 强度消减（Strength Reduction）**：
```
x * 2  →  x << 1
x / 2  →  x >> 1  （无符号整数）
x * x  →  x²
```
- 用更快的操作替代昂贵的操作
- 特别有效于循环中的乘法

**3. 死代码消除（Dead Code Elimination）**：
```
x = 1
y = x + 2
x = 3  // x 的第一次赋值是死的
```
- 算法：使用数据流分析（到达定义）
- 删除未使用的定义

**4. 公共子表达式消除（Common Subexpression Elimination）**：
```
t1 = b * c
...
t2 = b * c  // 用 t1 替换 t2
```
- 识别并复用已计算的相同表达式
- 使用可用表达式分析

**复杂度**：每个优化 O(n)，其中 n 是基本块长度。

**应用**：基础编译器优化、解释器优化。

### 全局优化

**定义**：跨基本块或整个函数的优化。

**1. 循环不变量外提（Loop Invariant Code Motion）**：
```
for i = 1 to 100:
  x = y + z  // y, z 在循环中不变
  a[i] = x
→
x = y + z  // 外提
for i = 1 to 100:
  a[i] = x
```
- 识别循环中不变的表达式
- 移到循环前置头
- 使用支配关系保证安全性

**2. 归纳变量（Induction Variable）**：
```
for i = 0 to 99:
  j = i * 4
  a[j] = 0
→
for j = 0 to 396 step 4:
  a[j] = 0
```
- 识别循环中的线性关系变量
- 用步长变量替代乘法

**3. 循环展开（Loop Unrolling）**：
```
for i = 0 to 99:
  a[i] = 0
→
for i = 0 to 96 step 4:
  a[i] = 0; a[i+1] = 0; a[i+2] = 0; a[i+3] = 0
处理剩余部分
```
- 增加指令级并行
- 减少循环开销
- 可能增加代码膨胀

**4. 循环交换（Loop Interchange）**：
```
for i = 1 to 100:
  for j = 1 to 100:
    a[i][j] = ...
→
for j = 1 to 100:
  for i = 1 to 100:
    a[i][j] = ...  // 更好的空间局部性
```
- 改善缓存局部性
- 适用于多维数组访问

**复杂度**：O(N + E) 数据流分析 + O(N) 优化。

**应用**：高性能编译器（GCC/LLVM）、数值计算优化。

### 寄存器分配

**定义**：将虚拟寄存器（无限）映射到物理寄存器（有限）。

**问题**：图着色
- 节点：虚拟寄存器
- 边：同时活跃的寄存器（干扰关系）
- 目标：用 k 种颜色（物理寄存器）着色，k ≥ 图的色数

**算法 1：线性扫描（Linear Scan）**：
```
按生命周期排序活跃区间：
  for interval in sorted order:
    if 可用寄存器非空:
      assign_reg(interval)
    else:
      spill 一个区间（优先生命周期的结束点最晚的）
      assign_reg(interval)
```
- **优点**：线性时间
- **缺点**：质量不如图着色

**复杂度**：O(n log n)，其中 n 是活跃区间数。

**算法 2：图着色（Graph Coloring - Chaitin）**：
```
1. 构建干扰图
2. 简化（Simplification）：
   repeat:
     选择度数 < k 的节点，入栈
   until 无此类节点
3. 选择（Spill）：
   if 栈为空:
     选择一个节点溢出到栈
     goto 2
4. 分配（Assignment）：
   while 栈非空:
     出栈节点，选择可用颜色
     if 无可用颜色:
       溢出到内存
```

**复杂度**：O(n²) 简化 + 分配。

**算法 3：迭代式寄存器合并（Iterated Register Coalescing - George/Appel）**：
- 在简化过程中尝试合并相关寄存器
- 平衡溢出成本和寄存器复制开销
- 质量优于 Chaitin

**溢出代码生成**：
```
溢出前：将寄存器保存到栈
溢出后：从栈加载到寄存器

例：
  t1 = a + b  // t1 需要溢出
→
  spill t1
  [spill_slot] = a + b
  t1 = [spill_slot]
```

**复杂度**：O(n²)。

**应用**：LLVM（线性扫描）、GCC（图着色）、JIT 编译器。

### 指令调度

**定义**：在不改变语义的前提下重新排序指令，提高并行性。

**目标架构**：
- **超流水线**：多个指令阶段重叠
- **超标量**：同时发射多条指令
- **VLIW**：编译器显式打包指令

**1. 列表调度（List Scheduling）**：
```
输入：DAG（依赖图），每节点的延迟
输出：调度序列

1. 计算每个节点的最早启动时间（est）和最晚启动时间（lst）
2. 维护就绪列表（所有前驱已调度的节点）
3. for cycle = 1 to MAX:
     for each slot:
       从就绪列表选择优先级最高的节点
       emit(node)
       更新就绪列表
```

**优先级函数**：
- **最迟启动时间**：lst（越小越优先）
- **关键路径长度**：到叶节点的最长路径
- **延迟**：执行时间长的优先

**2. 追踪调度（Trace Scheduling）**：
```
1. 识别热路径（trace，高频执行路径）
2. 对 trace 全局调度
3. 处理 trace 外的分支（补偿代码）
```
- 适用于 VLIW/超标量
- 可能增加分支预测失败的开销

**3. 软件流水（Software Pipelining）**：
```
原循环：
  for i = 0 to N-1:
    S1: load a[i]
    S2: compute
    S3: store b[i]

流水化后：
  i = 0
  load a[0]           // iteration 0
  load a[1]; compute b[0]
  load a[2]; compute b[1]; store b[0]
  ...
  compute b[N-1]; store b[N-2]
  store b[N-1]
```
- 循环展开 + 指令调度
- 隐藏指令延迟
- 需要处理序言和尾声代码

**复杂度**：O(n²) 列表调度，O(n·log n) 软件流水。

**应用**：VLIW 编译器（IA-64）、数字信号处理、GPU 编译。

---

## 代码生成

### 目标代码选择

**定义**：将中间表示映射到目标机器指令。

**算法 1：树模式匹配（Tree Pattern Matching）**：
```
输入：中间表示树
输出：指令序列

1. 每条目标指令定义一个树模式（瓦片，tile）
2. 使用动态规划找到最优覆盖：
   对于树中每个节点：
     cost[node] = min_{tile 覆盖 node} (tile.cost + Σ cost[child])
3. 从根节点回溯，选择最优 tile
```

**例**：
```
模式树：
  ADD      →  ADD r1, r2, r3    (cost: 1)
  LOAD     →  LOAD r1, [addr]   (cost: 1)
  CONSTANT →  MOV r1, #imm      (cost: 1)

表达式树：
      +
     / \
    *   3
   / \
  5   2

最优覆盖：
  MOV r1, #5
  MOV r2, #2
  MUL r3, r1, r2
  MOV r4, #3
  ADD r5, r3, r4
```

**算法 2：DAG 覆盖（DAG Covering）**：
```
对于 DAG（考虑公共子表达式）：
1. 自底向上计算每个节点的最优覆盖
2. 处理节点间共享的代价
```

**算法 3：窥孔优化（Peephole Optimization）**：
```
滑动窗口（通常 2-5 条指令）：
  pattern matching → replacement

例：
  MOV r1, r2
  MOV r2, r3
→
  MOV r1, r3
  // 删除冗余复制

  ADD r1, r2, #0
→
  // 删除无用加法

  JZ label
  JMP label
→
  JZ label
  // 删除不可达跳转
```

**复杂度**：树模式匹配 O(n)，DAG 覆盖 O(n²)。

**应用**：代码生成器生成（LLVM TableGen、GCC insn-attr）。

### 栈帧布局

**定义**：栈帧（Stack Frame）是函数调用时在栈上分配的内存块，存储局部变量、参数、返回地址等。

**栈帧结构（x86-64）**：
```
高地址
  | 调用者的栈帧 |
  +--------------+
  | 返回地址     |
  +--------------+
  | 保存的寄存器 |  ← 栈指针（%rbp 或 %rsp）
  +--------------+
  | 局部变量     |
  +--------------+
  | 参数传递区   |
低地址
```

**寄存器使用约定**：
- **调用者保存寄存器**（Caller-Saved）：调用前必须保存
- **被调用者保存寄存器**（Callee-Saved）：被调用者必须恢复

**算法（栈帧分配）**：
```
1. 活跃变量分析，确定每个变量的活跃范围
2. 将变量分配到栈偏移量
3. 对齐：对齐到栈的自然边界（如 16 字节）

for each variable v:
  offset[v] = current_sp
  current_sp += size(v)
  current_sp = align(current_sp, alignment(v))
```

**栈上寻址**：
```
x = y + z  // x 在 [rbp-8], y 在 [rbp-16], z 在 [rbp-24]

→
  MOV rax, [rbp-16]
  ADD rax, [rbp-24]
  MOV [rbp-8], rax
```

**叶函数优化**：
- 无调用其他函数
- 可省略返回地址和栈帧
- 直接使用寄存器参数

**复杂度**：O(n) 分配。

**应用**：x86/ARM/RISC-V 代码生成。

### 垃圾回收

**定义**：自动回收不再使用的内存对象。

**算法 1：标记-清除（Mark-Sweep）**：
```
标记（从根开始）：
  mark(obj):
    if obj 已标记: return
    obj.marked = true
    for each field f in obj:
      mark(obj.f)

清除（遍历堆）：
  for each obj in heap:
    if obj.marked:
      obj.marked = false
    else:
      free(obj)
```
- **优点**：实现简单
- **缺点**：产生内存碎片，暂停时间与堆大小成正比

**复杂度**：O(n)，其中 n 是堆中对象数。

**算法 2：复制（Copying）**：
```
堆分为两个半空间（from/to）：
  for each obj in from_space:
    if obj 从根可达:
      复制 obj 到 to_space
      更新引用到新地址
  交换 from/to
```
- **优点**：无内存碎片，分配快（指针递增）
- **缺点**：堆空间利用率减半

**复杂度**：O(n)，但实际只处理活对象。

**算法 3：引用计数（Reference Counting）**：
```
obj.ref_count++  // 新引用
obj.ref_count--  // 释放引用
if obj.ref_count == 0:
  free(obj)
  for each field f in obj:
    f.ref_count--
```
- **优点**：立即回收，无全局暂停
- **缺点**：无法处理循环引用，维护开销大

**复杂度**：每个赋值 O(1)。

**算法 4：分代 GC（Generational GC）**：
```
假设：大多数对象死得很快

分代结构：
  - 新生代（Young Generation）：Eden + Survivor
  - 老年代（Old Generation）

Minor GC：
  只扫描新生代
  幸存对象晋升到老年代

Major GC：
  扫描整个堆
  触发条件：老年代空间不足
```
- **优点**：减少 GC 暂停时间
- **缺点**：需要处理跨代引用（写屏障）

**写屏障（Write Barrier）**：
```
obj.field = new_value  // 写操作
if obj 在老年代, new_value 在新生代:
  记录跨代引用（卡表，Card Table）
```

**复杂度**：Minor GC O(活对象)，Major GC O(堆大小)。

**应用**：Java（G1/ZGC）、Go（三色标记）、JavaScript（V8）、Python（引用计数 + 分代 GC）。

---

## 链接与加载

### 静态链接与动态链接

**定义**：
- **静态链接**：在编译时将所有目标文件和库合并成可执行文件
- **动态链接**：运行时加载和链接共享库（.so/.dll/.dylib）

**静态链接过程**：
```
1. 符号解析（Symbol Resolution）：
   - 全局符号：定义和引用匹配
   - 外部符号：在其他目标文件中查找
   - 多重定义：报错或选择一个（弱符号）

2. 重定位（Relocation）：
   - 修改指令和数据的地址引用
   - 加载地址确定后更新

例（x86-64）：
  call printf  // 相对调用，需重定位
  .long buffer // 地址引用，需重定位
```

**动态链接过程**：
```
1. 编译：生成位置无关代码（PIC）
2. 链接：生成共享库和动态链接器信息
3. 运行：
   - 动态链接器（ld.so/dyld）加载共享库
   - 符号解析（延迟绑定，Lazy Binding）
   - 重定位（GOT/PLT）
```

**PLT（Procedure Linkage Table）**：
```
调用动态库函数：
  call printf@plt  →  跳转到 PLT[0]
  PLT[0]: 跳转到动态链接器解析地址
  解析后：PLT 条目直接跳转到实际函数
```

**复杂度**：静态链接 O(n)，动态链接 O(n) 首次 + O(1) 后续。

**应用**：Linux ELF、Windows PE、macOS Mach-O。

### ELF 格式

**定义**：Executable and Linkable Format（ELF）是 Linux/Unix 的可执行文件格式。

**ELF 结构**：
```
ELF Header
  Program Header Table（运行时加载）
  Section Header Table（链接/调试）

段（Segments，运行时视图）：
  .text：代码段
  .data：数据段
  .bss：未初始化数据段
  .rodata：只读数据段
  .interp：动态链接器路径
  .dynamic：动态链接信息
  .got：全局偏移表
  .plt：过程链接表

节（Sections，链接时视图）：
  .symtab：符号表
  .strtab：字符串表
  .rel.text：代码重定位表
  .rel.data：数据重定位表
  .debug：调试信息
```

**符号表（Symbol Table）**：
```
typedef struct {
    uint32_t st_name;      // 符号名（在 .strtab 的偏移）
    uint8_t  st_info;      // 符号类型和绑定
    uint8_t  st_other;     // 可见性
    uint16_t st_shndx;     // 所在节索引
    uint64_t st_value;     // 符号值（地址）
    uint64_t st_size;      // 符号大小
} Elf64_Sym;
```

**重定位表（Relocation Table）**：
```
typedef struct {
    uint64_t r_offset;     // 需重定位的位置
    uint32_t r_info;       // 符号索引 + 重定位类型
    int32_t  r_addend;     // 加数
} Elf64_Rela;

重定位类型（x86-64）：
  R_X86_64_64：64 位绝对地址
  R_X86_64_PC32：32 位 PC 相对地址
  R_X86_64_PLT32：PLT 相对调用
  R_X86_64_GLOB_DAT：GOT 条目重定位
```

**加载过程**：
```
1. 读取 ELF Header 和 Program Header Table
2. 将段映射到内存（mmap）
3. 重定位（加载时重定位，如果需要）
4. 初始化：执行 .init 段，调用构造函数
5. 跳转到入口点（entry point）
```

**动态链接数据结构**：
```
动态节（.dynamic）：
  DT_NEEDED：依赖的共享库
  DT_INIT：初始化函数
  DT_FINI：析构函数
  DT_STRTAB：符号字符串表
  DT_SYMTAB：符号表
  DT_REL：重定位表
  DT_JMPREL：PLT 重定位表（延迟绑定）
```

**复杂度**：O(n) 加载，O(n) 符号解析（n 是符号数）。

**应用**：Linux 可执行文件、共享库、目标文件。

### GOT 与 PLT

**定义**：
- **GOT（Global Offset Table）**：存储外部变量的地址
- **PLT（Procedure Linkage Table）**：存储跳转到外部函数的桩代码

**延迟绑定（Lazy Binding）**：
```
首次调用 printf@plt：
  PLT[1]: 指向 GOT[1]
  GOT[1]: 指回 PLT[1] + 6（解析函数）
  跳转到动态链接器，解析 printf，填充 GOT[1]

后续调用：
  PLT[1] → GOT[1] → printf（已解析）
```

**GOT 结构**：
```
GOT[0]: 存储动态链接器信息
GOT[1]: 链接映射（link map）
GOT[2]: 可用
GOT[3+]：外部变量的地址
```

**PLT 结构**：
```
PLT[0]: 标准桩（调用解析函数）
PLT[1+]: 每个动态链接函数一个条目

PLT[i]:
  JMP *GOT[i]  // 跳转到 GOT 条目
  PUSH i       // 推送重定位索引
  JMP PLT[0]   // 调用解析函数
```

**位置无关代码（PIC）**：
```
访问 GOT：
  call next_label
next_label:
  pop rax           // 获取当前地址
  add rax, offset_to_got  // 计算 GOT 地址
  mov rdi, [rax + i*8]    // 读取 GOT[i]
```

**复杂度**：首次调用 O(1) 解析，后续 O(1) 直接跳转。

**应用**：动态链接的共享库、位置无关可执行文件。

---

## 实际编译器

### GCC/Clang 编译流水线

**GCC 编译阶段**：
```
源代码（.c/.cpp）
  ↓ 预处理
预处理输出（.i）
  ↓ 编译
汇编代码（.s）
  ↓ 汇编
目标文件（.o）
  ↓ 链接
可执行文件
```

**各阶段详解**：

**1. 预处理**：
- 宏展开
- 文件包含（#include）
- 条件编译（#ifdef/#ifndef）
- 删除注释
- 输出：纯代码文件（.i）

**2. 编译**：
```
GCC 中间表示：
  1. GENERIC：AST 形式
  2. GIMPLE：三地址码形式（类似三地址码）
  3. RTL：寄存器传输语言（接近目标机器）
```
- 词法分析、语法分析
- 语义分析（类型检查）
- 优化（GIMPLE 优化）
- 输出：汇编代码（.s）

**3. 汇编**：
- 汇编代码转换为机器码
- 生成符号表、重定位信息
- 输出：目标文件（ELF/COFF/Mach-O）

**4. 链接**：
- 符号解析
- 重定位
- 合并段
- 输出：可执行文件

**Clang/GCC 区别**：
- **Clang**：基于 LLVM，前端（C/C++/Objective-C）
- **GCC**：完整的编译工具链（前端 + 中端 + 后端）

**复杂度**：整体 O(n)，各阶段 O(n)。

**应用**：C/C++/Objective-C/Fortran/Ada 等语言的编译。

### LLVM IR

**定义**：LLVM Intermediate Representation（LLVM IR）是 SSA 形式的中间表示，用于编译器优化。

**LLVM IR 特性**：
- **SSA 形式**：每个变量只赋值一次
- **类型化**：每个指令和操作数都有类型
- **无限虚拟寄存器**：%0, %1, %2, ...
- **显式控制流**：基本块 + 跳转指令

**LLVM IR 指令示例**：
```
; 函数定义
define i32 @add(i32 %a, i32 %b) {
entry:
  ; 二元操作
  %result = add i32 %a, %b
  ; 比较
  %is_zero = icmp eq i32 %result, 0
  ; 条件跳转
  br i1 %is_zero, label %then, label %else

then:
  ret i32 0

else:
  ; 函数调用
  call void @print(i32 %result)
  ret i32 %result
}
```

**LLVM 优化 Pass**：
```
内存优化：
  - 内存到寄存器提升（Mem2Reg）：栈变量 → SSA
  - 死存储消除（Dead Store Elimination）

循环优化：
  - 循环不变量外提（LICM）
  - 循环展开（Loop Unroll）
  - 循环向量化（Loop Vectorization）

数据流优化：
  - 常量传播（Constant Propagation）
  - 稀疏条件常量传播（SCCP）
  - 死代码消除（DCE）

内联：
  - 函数内联（Function Inlining）
  - 跨模块内联（LTO - Link Time Optimization）
```

**LLVM 后端**：
```
目标相关阶段：
  1. 指令选择（Instruction Selection）
  2. 寄存器分配（Register Allocation）
  3. 指令调度（Instruction Scheduling）
  4. 代码生成（Code Generation）

目标架构：
  - x86/x86-64
  - ARM/AArch64
  - RISC-V
  - PowerPC
  - MIPS
```

**LLVM 工具链**：
```
clang：前端编译器
llc：LLVM IR → 汇编
opt：LLVM IR 优化器
llvm-as：汇编 → LLVM bitcode
llvm-dis：LLVM bitcode → 汇编
lld：链接器
```

**复杂度**：IR 生成 O(n)，优化 O(n·log n)，代码生成 O(n)。

**应用**：Clang、Rust（rustc 使用 LLVM）、Swift、Kotlin/Native。

### JIT 编译

**定义**：即时编译（Just-In-Time Compilation）在运行时将字节码编译为机器码。

**JIT 编译器结构**：
```
解释器（Interpreter）
  ↓ 热点检测（Profiling）
JIT 编译器
  ↓ 代码生成
机器码
  ↓ 执行
优化（基于运行时信息）
```

**V8 JavaScript 引擎**：
```
执行流程：
  1. 解析器（Parser）→ 字节码
  2. 解释器（Ignition）执行字节码
  3. 热点检测：调用频率 > 阈值
  4. 优化编译器（TurboFan）→ 机器码
  5. 去优化（Deoptimization）：类型假设失败时回退

内联缓存（Inline Cache, IC）：
  - 缓存属性访问的位置
  - 多态内联缓存（Polymorphic IC）
  - 单态内联缓存（Monomorphic IC）

隐藏类（Hidden Classes）：
  - 动态添加属性时创建新类
  - 相同属性结构的对象共享类
  - 优化属性访问
```

**PyPy（Python JIT）**：
```
技术：
  - 元跟踪（Meta-Tracing）：解释解释器
  - RPython 语言：编写解释器，自动生成 JIT
  - 循环识别与优化

优点：
  - 比 CPython 快 5-10 倍
  - 自动 JIT，无需手工优化
```

**LLVM JIT（ORC）**：
```
On-Demand Compilation（ORC JIT）：
  1. 模块编译
  2. 符号解析
  3. 代码生成
  4. 内存映射
  5. 执行

lazy reexports: 延迟符号解析
  symbol resolution: 链接时解析符号
  memory management: 自动管理 JIT 代码内存
```

**JIT vs AOT（Ahead-Of-Time）**：
- **JIT**：
  - 优点：可利用运行时信息优化，启动快
  - 缺点：运行时编译开销，内存占用大
- **AOT**：
  - 优点：编译一次，多次执行，无运行时开销
  - 缺点：无法利用运行时信息，优化受限

**复杂度**：解释 O(n·e)，JIT 编译 O(n)，热点检测 O(n)。

**应用**：V8（Chrome/Node.js）、PyPy、GraalVM、Java HotSpot。

### WebAssembly

**定义**：WebAssembly（Wasm）是面向 Web 的二进制指令格式，用于高性能计算。

**WebAssembly 特性**：
- **二进制格式**：紧凑，快速解析
- **沙箱执行**：内存隔离，安全
- **线性内存**：单个连续内存空间
- **无垃圾回收**：手动管理（或语言自带 GC）
- **与 JS 互操作**：可调用 JS，JS 可调用 Wasm

**WebAssembly 模块结构**：
```
二进制格式（.wasm）：
  - Type Section：函数签名
  - Import Section：导入函数
  - Function Section：函数声明
  - Table Section：函数表（间接调用）
  - Memory Section：内存定义
  - Global Section：全局变量
  - Export Section：导出函数
  - Code Section：函数体
  - Data Section：数据段
  - Start Section：启动函数
```

**WebAssembly 指令集（栈虚拟机）**：
```
局部变量：
  local.get 0    // 压入局部变量 0
  local.set 1    // 弹出到局部变量 1
  local.tee 2    // 压入并设置局部变量 2

控制流：
  block          // 基本块
  loop           // 循环
  if             // 条件
  br             // 跳转
  br_if          // 条件跳转

算术：
  i32.add        // i32 栈顶两个相加
  i64.mul        // i64 栈顶两个相乘
  f32.div        // f32 除法
```

**编译到 WebAssembly**：
```
源语言（Rust/C++/Go）
  ↓ 前端（clang/LLVM）
LLVM IR
  ↓ 后端（LLVM Wasm backend）
WebAssembly
  ↓ 优化
优化后的 Wasm
```

**WebAssembly 运行时**：
```
浏览器引擎：
  - V8（Chrome/Edge）
  - SpiderMonkey（Firefox）
  - JavaScriptCore（Safari）

独立运行时：
  - Wasmtime
  - WasmEdge
  - Wasmer

特性：
  - AOT 编译：Wasm → 机器码
  - JIT 编译：热点函数优化
  - 内存安全：边界检查、类型检查
```

**WebAssembly 优化**：
```
常量折叠：
  i32.const 5
  i32.const 3
  i32.add
→
  i32.const 8

死代码消除：
  block
    i32.const 1
    drop
  end
→
  // 删除无用操作

循环展开：
  loop
    i32.const 1
    drop
    br 0
  end
→
  // 根据循环次数展开
```

**WebAssembly 与 JavaScript 互操作**：
```
Wasm 导出函数：
  (export "add" (func $add))

JavaScript 调用：
  const result = instance.exports.add(5, 3);

JavaScript 导入函数：
  (import "js" "log" (func $log (param i32)))

Wasm 调用：
  call $log  // 调用 JS 的 log 函数
```

**复杂度**：解析 O(n)，编译 O(n)，执行 O(n)。

**应用**：浏览器高性能计算、边缘计算、云计算。

---

## 总结

编译器是将高级语言转换为机器码的程序，涉及多个阶段：

1. **词法分析**：正则表达式 → NFA → DFA → 最小化
2. **语法分析**：CFG → 语法树（LL/LR 分析）
3. **语义分析**：符号表、类型检查、AST 构建
4. **中间表示**：三地址码、SSA、数据流分析
5. **代码优化**：局部/全局优化、寄存器分配、指令调度
6. **代码生成**：目标代码选择、栈帧布局、垃圾回收
7. **链接与加载**：静态/动态链接、ELF 格式、GOT/PLT

每个阶段都有明确的理论基础（定理、证明）和工程实现（算法、复杂度），确保编译器正确性、效率、可维护性。
