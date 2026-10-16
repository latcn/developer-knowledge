# 《Crafting Interpreters》第一部分完全知识体系

## ——从字符流到面向对象：jlox 解释器完整实现


### 🎯 导读：如何阅读这份文档

这是一份 **“概念-逻辑-代码”三维打通** 的知识地图。每一章节都遵循同一个底层模板：**公理 → 推论 → 矛盾/边界 → 实战落点**。如果你只记住了代码，你会被下一章冲垮；如果你只记住了概念，你写不出解释器。这份文档的目标是：**让你合上电脑，也能完整推演从 `var a = 1 + 2;` 到 `a` 被求值的全链路物理过程**。


## 第一章：扫描器（Scanning）——从字符流到 Token 流

### 🧠 底层公理

- A1：计算机只能**逐字符线性读取**源代码，无法“一眼扫过”。
- A2：编程语言由**有意义的语义单元（Token）**构成，编译器必须先识别出这些单元。
- A3：同一个字符在不同上下文中可能代表不同含义（如 `-` 既是一元负号也是二元减号）。

### 📐 逻辑推论链条

1. 逐字符线性读取 + 需要识别语义单元 → 扫描器必须有一个**循环**，从头到尾遍历每个字符，在恰当边界切分 Token。
2. 字符含义依赖上下文 + 需要判断当前字符属于哪种 Token → 扫描器必须能**往前看（Lookahead）**一两个字符。
3. 需要往前看 + 不同 Token 的规则不同 → 扫描逻辑必须按类型**分支处理**：单字符直接匹配，双字符（如 `!=`）需要试探性消费，长 Token（数字、字符串、标识符）需要专门的子程序完整读取。
4. 分支处理 + 需要输出语义单元 → 每识别完一个 Token，就将其（类型、字面值、行号）存入 **Token 列表**。
5. 需要处理无意义内容（空格、注释） + 需要错误恢复 → 扫描器跳过空格/注释（但更新行号），遇到非法字符时**报告错误但继续扫描**，以便一次性发现多个错误。
6. 关键字与标识符共享前缀（如 `or` vs `orchid`）→ 必须采用 **“最长匹配（Maximal Munch）”** 原则——能匹配更长的 Lexeme 就优先匹配更长的。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **Token** | 源代码的最小语义单元，包含类型、字面值和位置信息。 | `Token` 类（`TokenType`, `lexeme`, `literal`, `line`） |
| **Lexeme** | Token 在源代码中对应的原始字符子串。 | `lexeme` 字段 |
| **关键字（Keyword）** | 语言保留的、具有特殊语义的标识符（如 `if`, `while`, `var`）。 | `keywords` 哈希表（`Map<String, TokenType>`） |
| **标识符（Identifier）** | 用户自定义的变量名、函数名等。 | `IDENTIFIER` TokenType |
| **字面量（Literal）** | 源代码中的常量值（数字、字符串、布尔值、nil）。 | `Literal` Token，`value` 字段存储实际值 |
| **最长匹配（Maximal Munch）** | 能匹配更长的 Lexeme 就优先匹配更长的，以区分 `or` 和 `orchid`。 | `identifier()` 方法：持续吃字符直到遇到非字母数字 |
| **Lookahead** | 扫描器“偷看”下一个字符以判断当前字符属于哪种 Token。 | `peek()` 和 `peekNext()` 方法 |
| **错误恢复（Error Recovery）** | 遇到非法字符时报告错误但继续扫描，争取一次编译发现多个错误。 | `error()` 方法 + 继续循环 |

### ⚙️ 核心实践模块

```
Scanner 类的核心流程：

scanTokens():
  while (!isAtEnd()):
    start = current;
    char c = advance();
    switch (c):
      case '(': addToken(LEFT_PAREN); break;
      case ')': addToken(RIGHT_PAREN); break;
      // ... 单字符 Token
      case '!': addToken(peek() == '=' ? BANG_EQUAL : BANG); break;
      // ... 双字符 Token
      case '/': if (peek() == '/') { while (peek() != '\n') advance(); } else addToken(SLASH); break;
      case ' ':
      case '\r':
      case '\t': break;  // 忽略空白
      case '\n': line++; break;
      case '"': string(); break;  // 字符串专用子程序
      default:
        if (isDigit(c)) number();
        else if (isAlpha(c)) identifier();
        else error("Unexpected character.");
  return tokens;
```

### 🔥 常见认知陷阱与边界

1. **“按空格分割”的谬误**：扫描器不是按空格切分 Token，而是基于**字符类别的贪婪循环**——遇到字母就开始持续吃，直到吃到非字母数字才停。
2. **字符串报错行号**：未终止的字符串报错应指向**首行**（起始引号位置），而非最后一行。
3. **扫描器不做语义分析**：扫描器绝不判断 `+` 是加法还是连接，也不区分一元负号与二元减号——那是语法分析器的职责。
4. **状态机推翻无状态假设？**：不推翻。扫描器可以有**词法状态**（如“是否在括号内”、“缩进深度”），但不能有**语义状态**（如“这个变量是什么类型”）。


## 第二章：表示代码（Representing Code）——AST 与访问者模式

### 🧠 底层公理

- A1：源码具有**递归嵌套的语法结构**（表达式可无限嵌套，语句可无限包含）。
- A2：计算机执行代码时，需要一种**内存中的数据结构**来“持有”这种嵌套结构——平坦的 Token 序列丢失了嵌套信息。
- A3：对同一组数据结构，存在多种不同的操作（求值、打印、优化），且这些操作会持续增长。

### 📐 逻辑推论链条

1. 语法递归嵌套 + 需要内存中持有结构 → 代码的内存表示必须是**树形结构（抽象语法树，AST）**。
2. 需要树形结构 + Lox 表达式文法（`expression → literal | unary | binary | grouping`）→ 语法树的**节点类型与文法产生式一一对应**。
3. 节点类型多样 + 操作持续增长 → 如果采用传统 OOP（在基类中为每个操作定义虚方法），每新增一个操作就要修改所有节点类，违反开闭原则（OCP）。
4. 需要支持无限增长的操作 → 必须引入**访问者模式（Visitor Pattern）**——将操作（列）与数据结构（行）解耦。每个新操作只需新增一个 Visitor 实现类，无需修改任何已有节点类。
5. 节点类和 Visitor 接口之间存在大量重复样板代码 → 必须用**元编程（代码生成）**自动生成这些代码，避免手写错误并保持一致性。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **抽象语法树（AST）** | 丢弃括号等具体语法信息的树形结构，只保留表达式的嵌套本质。 | `Expr` 基类及子类（`Binary`, `Literal`, `Unary`, `Grouping`） |
| **解析树（Parse Tree）** | 完整保留所有文法细节（包括括号）的树。Lox 使用的是 AST，而非解析树。 | 未实现（Lox 丢弃括号信息） |
| **访问者模式（Visitor Pattern）** | 将操作从数据结构中解耦的设计模式。新增操作无需修改已有类。 | `Expr.Visitor<R>` 接口 + `accept(Visitor<R>)` 方法 |
| **表达式问题（Expression Problem）** | 扩展数据类型 vs 扩展操作的零和博弈。访问者模式擅长扩展操作，不擅长扩展数据类型。 | `Ternary` 新增需改所有 Visitor 实现 |
| **代码生成（Code Generation）** | 用元编程自动生成节点类和 Visitor 接口的样板代码，防止手写遗漏。 | `DefineAst.java`（主程序）生成 `Expr.java` |
| **多态（Polymorphism）** | 基类引用指向子类对象，通过虚方法动态分派。 | `Expr` 基类 + `Binary`/`Literal`/`Unary`/`Grouping` 子类 |

### ⚙️ 核心实践模块

```
代码生成器 DefineAst.java 的流程：

1. 定义产出目录和目标文件名（如 "Expr.java"）。
2. 定义所有节点类型及字段：
   - Binary: Expr left, Token operator, Expr right
   - Literal: Object value
   - Unary: Token operator, Expr right
   - Grouping: Expr expression
3. 为每个节点类型生成静态嵌套类：
   - 构造函数（字段初始化）
   - fields（public final）
   - accept(Visitor<R> visitor) 方法
4. 生成 Visitor<R> 接口：
   - 为每个节点类型声明 visitXxx(ClassName xxx) 方法。
5. 输出到文件。
```

### 🔥 常见认知陷阱与边界

1. **AST 丢弃括号是否丢失信息？**：没有。括号用于指导解析器如何构建树；一旦树建成，括号的使命即告终结。`1 + (2 * 3)` 和 `1 + 2 * 3` 生成完全相同的 AST。
2. **访问者模式是否完美解耦？**：不。它只是把“新增操作的代价”与“新增类的代价”进行了对调。新增类时，必须修改 Visitor 接口和所有现有实现——这是表达式问题的本质。
3. **`Object` 存放字面量是偷懒吗？**：不是，是动态类型语言的一脉相承。Lox 变量类型在运行时决定，AST 层强行用具体类型（如 `IntExpr`, `StringExpr`）反而与语言哲学相悖。
4. **`Break` 放在 Expr 还是 Stmt？**：必须在 `Stmt` 中，因为 `break` 没有返回值，且不能出现在需要值的上下文中。


## 第三章：解析表达式（Parsing Expressions）——Pratt 解析器

### 🧠 底层公理

- A1：解析器（Parser）是扫描器（Scanner）的**逆过程**——把平坦的 Token 序列重新组合成具有嵌套层次的树形结构。
- A2：上下文无关文法（CFG）本身**不包含优先级和结合性信息**——`binary → expression operator expression` 允许 `1-2-3` 被解析为 `(1-2)-3` 或 `1-(2-3)`。
- A3：递归下降解析器是文法规则的**“逐字翻译”**——每个产生式对应一个函数。

### 📐 逻辑推论链条

1. CFG 本身无优先级 + 需要消除歧义 → 必须把单一文法**分层**：每个优先级层级对应一个解析方法（`equality` → `comparison` → `term` → `factor` → `unary` → `primary`），或者采用 **Pratt 解析（优先级攀升）**。
2. Pratt 解析是分层文法的“压缩版” → 用一个带 `minPrecedence` 参数的 `parseExpression()` 函数 + `while` 循环替代 6 个独立函数。
3. 左操作数先解析 + 运算符序列（如 `1 - 2 - 3`）→ 自然形成**左结合**嵌套树（`(1-2)-3`）：将左操作数存入变量，然后不断用新的右操作数构建 `Binary` 节点并回存。
4. 右操作数调用下一层（更高优先级）方法 + 分层文法 → 自然形成**优先级**：高优先级的运算符在 AST 中处于更深的嵌套位置。
5. Pratt 解析器的“前缀”与“中缀”阶段 → 一元运算符和二元运算符的同名冲突（如 `-1` vs `3 - 4`）通过阶段划分解决：前缀阶段先吃，中缀阶段后处理。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **递归下降解析（Recursive Descent Parsing）** | 每个文法产生式对应一个函数，函数之间递归调用的解析方法。 | `expression()`, `equality()`, `comparison()`, `term()`, `factor()`, `unary()`, `primary()` |
| **Pratt 解析（Pratt Parsing / 优先级攀升）** | 用一个带 `minPrecedence` 参数的循环替代多层函数调用，通过数值比较控制优先级。 | `parseExpression(int minPrecedence)` + `while` 循环 |
| **前缀解析（Prefix Parse）** | 处理以当前 Token 开头的表达式（字面量、分组、一元运算符）。 | `parsePrefix()` 方法 |
| **中缀解析（Infix Parse）** | 处理在已有左操作数之后的运算符（二元运算符）。 | `parseInfix()` 方法（含在 `while` 循环中） |
| **左结合（Left Associativity）** | 相同优先级的运算符从左到右计算（`1-2-3` → `(1-2)-3`）。 | 递归调用右侧时传入 `precedence + 1` |
| **右结合（Right Associativity）** | 相同优先级的运算符从右到左计算（`a = b = 3` → `a = (b = 3)`）。 | 递归调用右侧时传入 `precedence`（不加 1） |
| **恐慌模式错误恢复（Panic Mode Error Recovery）** | 遇到错误时抛出异常，跳过 Token 直到找到同步点（如分号或语句关键字），然后继续解析。 | `parseError()` + `synchronize()` 方法 |

### ⚙️ 核心实践模块（Pratt 解析器核心循环）

```
parseExpression(minPrecedence):
  left = parsePrefix()   // 解析左操作数（字面量、分组、一元）
  while (true):
    op = peek()
    if op 不是中缀运算符: break
    if getPrecedence(op) < minPrecedence: break
    consume(op)
    right = parseExpression(getPrecedence(op) + (isLeftAssoc ? 1 : 0))
    left = new Binary(left, op, right)
  return left
```

### 🔥 常见认知陷阱与边界

1. **左结合 vs 右结合**：右结合不是靠“设置优先级”，而是靠递归传参时**不加 1**。
2. **一元 vs 二元识别**：不靠“提前多看 Token”，而是靠“前缀阶段先吃”。当前 Token 是 `-` 时，若它出现在表达式开头（左侧无东西），直接作为一元处理；若左侧已有解析好的左操作数，则作为二元处理。
3. **`1 - * 2` 的错误恢复**：`-` 是二元运算符，右侧的 `*` 不能作为表达式开头。解析器抛出 ParseError，**吞掉** `*` Token，返回占位符，然后尝试解析 `2`，最终得到 `1 + 2`（丢弃了 `*`）并附带报错信息。
4. **Pratt vs 传统递归下降**：Pratt 没有推翻“文法即代码”的公理，它只是用“带数值参数的循环”替代了“显式的函数调用栈深度”，两者等价。


## 第四章：求值表达式（Evaluating Expressions）——解释器的“大脑”

### 🧠 底层公理

- A1：解释器执行的是**树形结构（AST）**，而非线性代码——从根节点开始，递归地访问每个节点。
- A2：每个 AST 节点知道“怎么计算自己”，但**计算逻辑与节点本身是分离的**（通过访问者模式）。
- A3：Lox 是**动态类型语言**——变量的类型在运行时决定，运算符的行为取决于操作数的运行时类型。

### 📐 逻辑推论链条

1. 执行 AST + 求值逻辑与节点分离 → 解释器通过访问者模式实现**后序遍历（Post-order Traversal）**——先递归求值子节点，再对结果应用当前节点的运算。
2. 动态类型需要运行时检查 + 具体运算符有不同的类型要求 → 二元运算符求值分两步：① 递归求值左右操作数；② 根据操作数的运行时类型决定执行哪种运算。`+` 检查数字（加法）或字符串（拼接）；`>` 要求数字；`==` 接受任意类型。
3. 后序遍历 + 需要处理类型错误 → 类型错误必须在运行时抛出 **RuntimeError**（携带 Token 信息），而非依赖 Java 的 `ClassCastException`，以便精确定位到用户代码的行号。
4. 运行时错误需要优雅处理 + REPL 需要持续运行 → 解释器的 `interpret()` 方法用 `try-catch` 包裹求值过程，捕获 RuntimeError 后报告错误并继续运行。
5. 求值逻辑在 Interpreter 中 + 需要向用户展示结果 → `interpret()` 调用 `evaluate()` 求值，再用 `stringify()` 将 Lox 值转换为用户友好的字符串（如 `3.0` → `3`）。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **后序遍历（Post-order Traversal）** | 先求值子节点，再求值父节点的遍历顺序。 | `visitBinaryExpr` 先 `evaluate(left)`，再 `evaluate(right)`，最后做运算 |
| **短路求值（Short-circuiting）** | `and` 和 `or` 根据左值决定是否求值右操作数。 | `visitLogicalExpr` 中先求值左操作数，若结果可决定则**直接返回**，跳过右操作数 |
| **运行时类型检查** | 在求值过程中用 `instanceof` 检查操作数类型，决定执行哪种操作。 | `visitBinaryExpr` 中的 `if (left instanceof Double && right instanceof Double)` |
| **RuntimeError** | 运行时发生的类型错误或未定义变量错误，携带 Token 信息以便定位。 | `RuntimeError` 类（继承自 `RuntimeException`） |
| **`stringify()`** | 将 Lox 值转换为用户友好的字符串表示。数字 `3.0` → `"3"`，`nil` → `"nil"`。 | `stringify(Object value)` 方法 |
| **表达式语句（Expression Statement）** | 将表达式包装成语句，使其能独立成行（如 `1 + 2;`），以便执行有副作用的表达式（如函数调用）。 | `Stmt.Expression` 节点 + `visitExpressionStmt` 实现 |
| **类型策略模式（Type Strategy Pattern）** | 将类型检查逻辑抽象为独立策略，避免 `instanceof` 爆炸。 | 你提出的架构改进方案（Lox 未使用） |

### ⚙️ 核心实践模块

```
Interpreter 类核心流程：

evaluate(Expr expr):
  return expr.accept(this);  // 访问者模式入口

visitBinaryExpr(expr):
  left = evaluate(expr.left)
  right = evaluate(expr.right)
  switch (expr.operator.type):
    case PLUS:
      if (left is Double and right is Double): return (Double)left + (Double)right
      if (left is String and right is String): return (String)left + (String)right
      throw RuntimeError("Operands must be two numbers or two strings.")
    case MINUS:
      checkNumberOperands(left, right)
      return (Double)left - (Double)right
    // ... 其他运算符

visitLogicalExpr(expr):
  left = evaluate(expr.left)
  if (expr.operator.type == OR):
    if (isTruthy(left)) return left;  // 短路：直接返回，不 evaluate(right)
  else: // AND
    if (!isTruthy(left)) return left;  // 短路：直接返回，不 evaluate(right)
  return evaluate(expr.right)

interpret(program):
  try:
    value = evaluate(program)
    print(stringify(value))
  catch (RuntimeError e):
    reportError(e)
```

### 🔥 常见认知陷阱与边界

1. **短路求值打破后序遍历**：`and` 和 `or` 不是函数调用，它们是**控制流结构**。`visitLogicalExpr` 先求值左操作数，若结果可决定，则**直接返回**，完全跳过 `evaluate(right)` 的调用。
2. **`nil` 的多语境规则**：`nil` 在算术运算中非法，但在逻辑运算中合法（视为假）。这是语言设计的**契约**，而非 bug。
3. **浮点数精度**：Lox 使用 Java 的 `Double`（IEEE 754），`1/3` 是近似值。Lox 不将其视为“错误”，而是工程折衷。`stringify` 检查“如果数值是整数（如 `1.0`），打印 `1`；否则打印原始双精度字符串”。


## 第五章：语句与状态（Statements and State）——变量的“记忆”

### 🧠 底层公理

- A1：编程的本质是组合与抽象，这需要 **“命名”** 的能力——变量是组合的基石。
- A2：语句（Statement）存在的意义在于产生 **“副作用（Side Effect）”**——它们不返回值，但能修改变量、产生输出。
- A3：变量需要 **“环境（Environment）”** 来存储和管理——解释器必须有“记忆”能力。

### 📐 逻辑推论链条

1. 需要命名能力 + 语句产生副作用 → 变量**声明**（`var a = 1;`）必须作为**语句**实现，而非表达式。声明变量的目的是引入一个新绑定（副作用），而非计算一个值。
2. 语句产生副作用 + 表达式本身不产生副作用 → 需要**表达式语句（Expression Statement）**，把表达式包装成语句，以便执行有副作用的表达式（如函数调用）。
3. 需要存储变量 + 作用域需要嵌套（如代码块）→ 环境必须是**可嵌套的链式结构**，每个环境持有一个指向其外层环境的引用（`enclosing`）。
4. 环境是链式的 + 变量查找需从内向外 → 变量访问时，从当前环境开始沿链向上查找，直到找到或到达全局环境。赋值同理。
5. 表达式语句 + 程序由语句列表构成 → 程序顶层就是一系列语句。解析器起始规则变为 `program → statement* EOF`。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **环境（Environment）** | 存储变量名到值的映射表，带 `enclosing` 指针实现链式作用域。 | `Environment` 类（`Map<String, Object>` + `enclosing`） |
| **变量声明（Variable Declaration）** | 使用 `var` 关键字在当前环境创建新绑定。 | `Stmt.Var` 节点 + `visitVarStmt` 调用 `environment.define(name, value)` |
| **变量赋值（Variable Assignment）** | 使用 `=` 运算符沿环境链向上查找并修改已有绑定。 | `Expr.Assign` 节点 + `visitAssignExpr` 调用 `environment.assign(name, value)` |
| **代码块（Block）** | 使用 `{ ... }` 创建新的嵌套作用域。 | `Stmt.Block` 节点 + `visitBlockStmt` 创建新 `Environment` |
| **表达式语句（Expression Statement）** | 将表达式包装成语句，使其能独立成行。 | `Stmt.Expression` 节点 + `visitExpressionStmt` 求值并丢弃结果 |
| **隐式全局声明（Implicit Global Declaration）** | 对不存在的变量赋值时，Lox 会在全局环境自动创建它。 | `assign()` 方法查到环境链顶端仍未找到时，调用 `global.define(name, value)` |
| **作用域嵌套（Lexical Nesting）** | 内部作用域可以访问外部作用域的变量，反之不行。 | `new Environment(previous)` 构造函数 |

### ⚙️ 核心实践模块

```
Environment 类核心逻辑：

class Environment {
  private final Map<String, Object> values = new HashMap<>();
  final Environment enclosing;

  Environment() { enclosing = null; }  // 全局环境
  Environment(Environment enclosing) { this.enclosing = enclosing; }  // 嵌套环境

  void define(String name, Object value) {
    values.put(name, value);  // 仅当前环境插入
  }

  Object get(Token name) {
    if (values.containsKey(name.lexeme)) return values.get(name.lexeme);
    if (enclosing != null) return enclosing.get(name);
    throw new RuntimeError("Undefined variable '" + name.lexeme + "'.");
  }

  void assign(Token name, Object value) {
    if (values.containsKey(name.lexeme)) {
      values.put(name.lexeme, value);
      return;
    }
    if (enclosing != null) {
      enclosing.assign(name, value);
      return;
    }
    // 隐式全局声明（Fallback）
    define(name.lexeme, value);
  }
}

visitBlockStmt:
  Environment previous = environment;
  environment = new Environment(previous);  // 进入代码块
  try { execute(stmt.statements); }
  finally { environment = previous; }  // 退出代码块，丢弃子环境
```

### 🔥 常见认知陷阱与边界

1. **`var a;` vs `var a = nil;`**：两者完全等价。`var a;` 是语法糖，解释器只是将 `a` 绑定为 `nil`。这不算“未初始化”，而是“已声明，值为 `nil`”。
2. **隐式全局声明的隐患**：Lox 允许 `a = 5;`（未声明直接赋值）在全局创建变量。这方便了 REPL 调试，但会掩盖拼写错误（如 `ab = 5` 本意是修改 `a`）。
3. **块级作用域的“环境瞬移”**：进入代码块时替换 `environment` 指针，退出时恢复。内部的 `a` 在恢复后彻底丢失（若无闭包引用），被 GC 回收。
4. **环境链只负责变量查找**：环境链不负责执行流控制。函数返回时，环境链的恢复和 Java 调用栈的弹出是**并行但独立**的两个机制。


## 第六章：控制流（Control Flow）——分支与循环

### 🧠 底层公理

- A1：程序的执行路径可以是**非线性的**——根据运行时条件跳过代码（`if`）或重复执行代码（`while`、`for`）。
- A2：控制流结构本身就是“副作用的生产者”——它们通过“**不执行**”某些代码来产生效果。
- A3：循环可以将有限的源代码转化为**无限（或任意长）**的执行时间——这是图灵完备性的核心标志。

### 📐 逻辑推论链条

1. 执行路径非线性 + 需要在运行时决定“走哪条路” → `if` 语句必须**打破“无条件求值所有子节点”**的规则：先求值条件，然后根据结果**只执行** `thenBranch` 或 `elseBranch` 中的一个。
2. 控制流是语句 + 需要表示“条件 + 两个分支” → `if` 的 AST 节点包含三个字段：`condition`、`thenBranch`、`elseBranch`（可为 `null`）。
3. 递归解析分支 + `else` 子句可选 + 嵌套 `if` → 出现 **“悬空 else（Dangling Else）”** 歧义。Lox 采用约定：`else` 总是绑定到最近的未匹配 `if`。
4. 循环产生无限执行 + 需要在每次迭代前检查条件 → `while` 循环的解释器实现**直接使用宿主语言（Java）的 `while` 循环**——每次迭代先求值条件，若为真则执行循环体，然后重复。
5. `while` 是基本循环 + `for` 循环是语法糖 → 解析器在解析 `for` 时，**不创建新的 AST 节点**，而是直接构造出等价的 `Block` + `While` 节点（脱糖 / Desugar）。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **`if` 语句（Conditional Branch）** | 根据条件值选择执行两个分支之一。 | `Stmt.If` 节点 + `visitIfStmt` 实现 |
| **悬空 else（Dangling Else）** | 嵌套 `if` 中，`else` 与哪个 `if` 匹配的歧义。 | `ifStatement()` 通过“急切匹配”解决：遇到 `else` 立即绑定到最近 `if` |
| **`while` 循环（Indefinite Loop）** | 条件为真时反复执行循环体。 | `Stmt.While` 节点 + `visitWhileStmt` 用 Java `while` 循环实现 |
| **`for` 循环（Definite Loop）** | 包含初始化、条件、增量的循环结构。 | Lox 解析器将 `for` **脱糖**为 `while` + `Block` |
| **脱糖（Desugaring）** | 在解析阶段将语法糖（如 `for`）转换为更基础的结构（如 `while` + `Block`）。 | `forStatement()` 方法 |
| **有限循环 vs 无限循环** | 有限循环的迭代次数可预测；无限循环（`while (true)`）永不结束，是图灵完备性的体现。 | 解释器将循环控制权完全交给程序员和操作系统 |
| **作用域泄漏（Scope Leak）** | `if (true) var a = 1;` 中 `a` 声明在当前环境，不被 `if` 隔离。 | Lox 允许单条语句不创建新环境，是“便利性”与“隔离性”的取舍 |

### ⚙️ 核心实践模块

```
visitIfStmt:
  if (isTruthy(evaluate(stmt.condition))) {
    execute(stmt.thenBranch);
  } else if (stmt.elseBranch != null) {
    execute(stmt.elseBranch);
  }

visitWhileStmt:
  while (isTruthy(evaluate(stmt.condition))) {
    execute(stmt.body);
  }

forStatement Desugar（脱糖）:
  for (var i = 0; i < 10; i = i + 1) print i;
  → 转换为：
  {
    var i = 0;
    while (i < 10) {
      print i;
      i = i + 1;
    }
  }
```

### 🔥 常见认知陷阱与边界

1. **`if` 不加花括号的作用域泄漏**：Lox 允许单条语句作为分支，不会创建新环境，导致变量泄漏到外部。这是语言设计者**有意为之**（保持解析器简单），而非 bug。
2. **无限循环责任归属**：解释器没有“步数限制”或“超时机制”。防止程序卡死是**程序员的责任**，解释器只保证语义正确性。这符合停机问题不可判定性的理论结论。
3. **脱糖的代价**：`for` 脱糖为 `while` 后，丢失了“初始化/条件/增量”的**结构化信息**，使调试器无法区分循环体与增量更新，也使 JIT 编译器无法将 `for` 优化为高效计数循环。


## 第七章：函数（Functions）——调用的“栈”

### 🧠 底层公理

- A1：函数是**一等公民**——可以被当作值来传递、存储和调用。
- A2：每次函数调用都必须创建一个**全新的、独立的局部环境**——递归要求每个调用帧拥有独立的参数绑定。
- A3：函数调用是 **“时空跳跃”**——执行流必须能穿透任意深度的嵌套，然后精确返回。

### 📐 逻辑推论链条

1. 函数是一等公民 + 调用语法允许任意表达式作为被调用者 → 函数调用的解析必须允许任意表达式作为 `callee`。解析器先解析一个 `primary` 表达式作为 `callee`，然后在一个 `while` 循环中反复匹配 `(`，每匹配到一个就调用 `finishCall()` 解析参数列表。
2. 每次调用创建独立环境 + 参数是函数的私有财产 → `LoxFunction.call()` 的实现分三步：① 以全局环境为父环境创建新的局部环境；② 形参与实参绑定；③ `executeBlock()` 执行函数体。
3. 每次调用独立环境 + 递归需要多个环境同时存在 → 递归调用的正确性依赖于**每个调用帧拥有独立的参数绑定**。`call()` 每次被调用都 `new Environment()`，旧环境挂载在调用栈上等待恢复。
4. `return` 需要穿透嵌套 + 访问者模式是递归遍历 → 树遍历解释器中，`return` 必须能穿透任意深度的访问者调用栈。Lox 用**抛出异常**实现：`Return` 异常向上冒泡，直到 `LoxFunction.call()` 的 `try-catch` 捕获。
5. 函数是值 + 函数声明是“创建值 + 绑定到变量” → 函数声明对应两步：① 创建 `LoxFunction` 对象；② 在当前环境中用函数名作为键存入。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **一等函数（First-class Functions）** | 函数可以作为值传递、存储在变量中、作为参数传递、作为返回值返回。 | `LoxFunction` 实现 `LoxCallable` 接口 |
| **LoxCallable 接口** | 所有可调用对象的统一接口，包含 `call()` 和 `arity()` 方法。 | `LoxFunction`、`ClockFunction`（原生函数）均实现此接口 |
| **调用栈（Call Stack）** | Java 方法的运行时栈帧堆栈，负责**执行流控制**（函数执行完回到哪里）。 | JVM 维护的调用栈 |
| **环境链（Environment Chain）** | `Environment` 的 `enclosing` 链表，负责**变量查找**（`a` 指向哪个值）。 | `environment.enclosing` 指针链 |
| **绑定方法（Bound Method）** | 将 `this` 绑定到特定实例的函数对象。 | `LoxFunction.bind(instance)` 创建新闭包环境 |
| **原生函数（Native Function）** | 在宿主语言（Java）中实现、暴露给 Lox 的内置函数。 | `ClockFunction` 实现 `LoxCallable` |
| **Return 异常（Return Exception）** | 用 Java 异常机制实现跨栈帧控制流转移。 | `Return` 类（继承 `RuntimeException`） |

### ⚙️ 核心实践模块

```
LoxFunction.call():
  Environment env = new Environment(interpreter.globals);
  for (int i = 0; i < declaration.params.size(); i++) {
    env.define(declaration.params.get(i).lexeme, arguments.get(i));
  }
  try {
    interpreter.executeBlock(declaration.body, env);
  } catch (Return e) {
    return e.value;  // 捕获 return 异常，返回其值
  }
  return null;  // 函数无返回时返回 nil

visitReturnStmt:
  Object value = null;
  if (stmt.value != null) value = evaluate(stmt.value);
  throw new Return(value);  // 抛出异常，穿透所有调用栈

visitFunctionStmt:
  LoxFunction function = new LoxFunction(stmt.function, environment);
  environment.define(stmt.function.name.lexeme, function);
```

### 🔥 常见认知陷阱与边界

1. **环境链 vs 调用栈的混淆**：环境链是**数据结构**（堆内存），负责变量查找；调用栈是**运行时机制**（JVM 栈帧），负责执行流控制。`return` 异常的传播**不依赖环境链**，而是依赖 Java 的调用栈。
2. **顶级 `return` 的崩溃**：全局 `return;` 没有 `LoxFunction.call()` 的 `try-catch` 捕获，会导致 REPL 崩溃。需在解析阶段禁止或在解释阶段忽略。
3. **函数重新声明的行为**：`fun foo() {}` 本质是 `var foo = LoxFunction(...);`。第二次 `fun foo() {}` 会覆盖环境中的 `foo` 键。旧对象若被其他变量引用，依然存活。
4. **原生函数的抽象纯洁性**：`visitCallExpr` 不区分 `LoxFunction` 和 `ClockFunction`，都通过 `LoxCallable` 接口调用。这种解耦使扩展原生函数极其简单。


## 第八章：解析与绑定（Resolving and Binding）——变量解析的“透视眼”

### 🧠 底层公理

- A1：词法作用域是程序的**静态属性**——仅凭阅读源代码就能确定变量指向哪个声明。
- A2：变量引用遵循 **“最近匹配”** 原则——它指向**前文中、最近的、同名的声明**。
- A3：静态分析是**在执行之前对代码“预读”**的过程——只分析结构，不产生副作用，不受控制流影响。

### 📐 逻辑推论链条

1. 词法作用域是静态的 + 解释器的环境查找存在 Bug → Bug 根源：环境是**可变哈希表**，闭包捕获的是**环境的引用**，而非快照。函数声明后、同一作用域内新增的变量会被闭包“意外”看到。
2. 需要修复 Bug + 不想大改现有代码 → 修复方案不是改用持久化数据结构，而是**在执行前进行一次完整的变量解析**，计算出每个变量引用在环境链中的“距离（Distance）”。
3. 在执行前完成变量解析 + 静态分析与动态执行独立 → 新增独立的 **`Resolver` 类**，在 Parser 生成 AST 之后、Interpreter 执行之前，**单独遍历一次 AST**。
4. `Resolver` 独立遍历 AST + 作用域是栈结构 → `Resolver` 内部维护 `scopes` 栈（`List<Map<String, Boolean>>`）。进入代码块时 `push`，离开时 `pop`。变量声明先 `declare`（标记 `false`），解析初始化器后再 `define`（标记 `true`），禁止自身引用。
5. `Resolver` 用栈记录作用域 + 遍历时发现变量引用 → `Resolver` 从 `scopes` 栈的**最内层向外查找**，找到后计算**该变量所在的作用域到当前作用域的距离**。存入 `Interpreter.locals` 映射表（键是 AST 节点，值是距离）。
6. `Resolver` 记录“距离” + 解释器执行时直接使用 → 解释器执行变量访问时，直接从 `locals` 中查出距离，用 `environment.getAt(distance, name)` **一步到位**获取值。若 `locals` 中无记录，则视为全局变量。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **词法作用域（Lexical Scope）** | 变量的可见范围由源代码中花括号的物理位置决定。 | Lox 的默认作用域规则 |
| **变量解析（Variable Resolution）** | 在执行前计算出每个变量引用指向哪个声明，以及它距离当前环境有多远。 | `Resolver` 类 |
| **距离（Distance）** | 变量所在环境到当前环境需要跨越的 `enclosing` 层数，是一个**编译期常量**。 | `locals` 映射表中的 `Integer` 值 |
| **声明（Declare）vs 定义（Define）** | `declare`：将变量标记为“未就绪”（`false`）；`define`：标记为“已就绪”（`true`）。 | `scopes` 栈中的 `Map<String, Boolean>` |
| **时间死区（Temporal Dead Zone）** | 从 `declare` 到 `define` 之间，变量已声明但不可访问，禁止自身引用（`var a = a;`）。 | 通过 `declare` + `define` 分离实现 |
| **闭包环境捕获（Closure Environment Capture）** | 函数对象持有“诞生时”的环境引用，使得闭包可以访问被捕获的自由变量。 | `LoxFunction` 的 `environment` 字段 |
| **全局 Fallback** | `Resolver` 未在局部栈中找到的变量，视为全局变量，运行时在全局环境中查找。 | `locals` 中无记录时的 Fallback 逻辑 |

### ⚙️ 核心实践模块

```
Resolver 核心流程：

visitBlockStmt:
  beginScope()  // push new Map<String, Boolean>
  resolve(stmt.statements)
  endScope()    // pop

visitVarStmt:
  declare(stmt.name)  // scopes.peek().put(name, false)
  if (stmt.initializer != null) {
    resolve(stmt.initializer)  // 解析初始化器
  }
  define(stmt.name)   // scopes.peek().put(name, true)

visitVariableExpr:
  if (!scopes.isEmpty()) {
    for (int i = scopes.size() - 1; i >= 0; i--) {
      if (scopes.get(i).containsKey(name)) {
        if (!scopes.get(i).get(name)) error("Cannot read local variable in its own initializer.");
        interpreter.locals.put(expr, scopes.size() - 1 - i);
        return;
      }
    }
  }
  // 未在局部栈中找到 → 标记为全局变量（不存储在 locals 中）

Interpreter.getAt:
  Object getAt(int distance, String name) {
    Environment env = environment;
    for (int i = 0; i < distance; i++) {
      env = env.enclosing;
    }
    return env.get(name);
  }
```

### 🔥 常见认知陷阱与边界

1. **`if(false)` 中的变量声明**：`Resolver` 会扫描 `thenBranch` 中的 `var a` 并记录在作用域栈中。但运行时条件为 `false`，`a` 未被创建。`Resolver` 不报错，因为它只负责**词法可见性**，不负责**初始化确定性**。报错发生在运行时（`RuntimeError: Undefined variable`）。
2. **全局变量的特殊性**：`Resolver` 不将全局作用域压入 `scopes` 栈，因此全局变量无 `distance`，在 `locals` 中无记录，运行时通过全局环境 Fallback 查找。
3. **`distance` 是静态计算的**：`distance` 是 `Resolver` 在遍历时计算的**编译期常量**，与运行时状态无关。闭包函数持有“诞生时”的环境快照，使得静态距离在任意时刻调用都能精准定位到被捕获的变量。


## 第九章：类（Classes）——数据的“胶囊”

### 🧠 底层公理

- A1：对象是数据（字段）与行为（方法）的 **“胶囊”**——将状态和操作捆绑在同一个命名空间下。
- A2：字段是**动态的、松散的**——类声明不定义字段，字段在运行时通过赋值“生长”出来。
- A3：方法是 **“绑定”到实例的函数**——方法被访问时，会捕获调用它的实例（`this`），形成一个闭包。

### 📐 逻辑推论链条

1. 对象是数据+行为的胶囊 + 需要创建新实例 → 类的实例化通过“调用”类对象本身实现（`Bagel()`），避免引入 `new` 关键字。`LoxClass` 实现 `LoxCallable` 接口。
2. 需要读写字段 + 需要语法支持 → 字段访问使用 `.` 语法：`instance.field` 是 get 表达式，`instance.field = value` 是 set 表达式。
3. 方法是绑定到实例的函数 + 字段和方法共享相同的命名空间 → 属性查找顺序：**先查字段，再查方法**。字段可以“遮蔽”同名方法。
4. 方法是绑定到实例的函数 + `this` 需要在方法体内指向当前实例 → `this` 被实现为**隐式声明的局部变量**，在方法被访问时绑定到当前实例。Resolver 在进入类的方法解析前，向作用域栈中压入包含 `this` 的隐式作用域。
5. `this` 是隐式局部变量 + 原始函数（`LoxFunction`）已持有环境闭包 → `visitGetExpr` 查找属性发现是方法时，创建一个新的 `LoxFunction` 对象，其闭包环境绑定了 `this`，父环境是原方法的闭包环境。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **类（Class）** | 用户自定义的复合数据类型，包含方法表。 | `LoxClass` 类（实现 `LoxCallable`） |
| **实例（Instance）** | 类的具体对象，持有字段的动态映射表。 | `LoxInstance` 类（持有 `Map<String, Object> fields`） |
| **字段（Field）** | 实例自身存储的属性，在运行时动态添加。 | `LoxInstance.fields` 中的键值对 |
| **方法（Method）** | 类中定义的函数，被该类的所有实例共享。 | `LoxClass.methods` 中的 `Map<String, LoxFunction>` |
| **绑定方法（Bound Method）** | 捕获了 `this` 实例的函数对象，在访问 `instance.method` 时创建。 | `LoxFunction.bind(instance)` |
| **`this` 隐式局部变量** | 在方法体内指向当前实例的特殊变量，由 Resolver 作为隐式作用域处理。 | `visitThisExpr` 将 `this` 解析为局部变量 |
| **`init` 构造函数** | 类中名为 `init` 的特殊方法，在 `LoxClass.call()` 中自动执行，可覆盖返回值。 | `LoxClass.call()` 中的 `try-catch` 捕获 `Return` 异常 |

### ⚙️ 核心实践模块

```
LoxClass.call():
  LoxInstance instance = new LoxInstance(this);  // 创建实例
  LoxFunction initializer = findMethod("init");
  if (initializer != null) {
    try {
      initializer.bind(instance).call(...);  // 执行 init，绑定 this
    } catch (Return returnValue) {
      if (returnValue.value != null) return returnValue.value;  // 允许劫持返回值
    }
  }
  return instance;  // 默认返回实例

LoxInstance.get():
  if (fields.containsKey(name)) return fields.get(name);  // 字段优先
  LoxFunction method = klass.findMethod(name);
  if (method != null) return method.bind(this);  // 返回绑定方法
  throw new RuntimeError("Undefined property '" + name + "'.");

LoxInstance.set():
  fields.put(name, value);  // 直接在字段映射表中插入或覆盖

visitClassStmt:
  LoxClass klass = new LoxClass(stmt.name.lexeme, superclass, stmt.methods);
  environment.define(stmt.name.lexeme, klass);
```

### 🔥 常见认知陷阱与边界

1. **字段遮蔽方法**：`instance.field = "string"` 会覆盖同名方法，导致 `instance.method()` 运行时崩溃。这是 Lox “字段优先”查找策略的直接后果。
2. **`init` 的返回值劫持**：`init` 方法可以通过 `return something;` 劫持构造函数返回值，丢弃新创建的实例。这实现了工厂模式，但破坏了类型一致性。
3. **绑定方法持有实例引用**：`fn = object.method;` 创建绑定方法，内部持有 `object` 的引用，因此 `object` 无法被 GC 回收，直到 `fn` 被销毁。


## 第十章：继承（Inheritance）——进化的“阶梯”

### 🧠 底层公理

- A1：继承的本质是 **“方法的自动委派”**——子类“借”用父类的方法，无需重新实现。
- A2：`super` 是一个**静态的（Static）**语法构造——它指向“定义当前方法的那个类的父类”，而不是“运行时对象的父类”。
- A3：继承链上**不能有环**——一个类不能直接或间接地继承自自身。

### 📐 逻辑推论链条

1. 方法自动向上委派 + 需要在查找方法时“向上爬” → `LoxClass.findMethod()` 先查自己的方法表，查不到就递归调用 `superclass.findMethod()`，直到继承链顶端。
2. `super` 是静态的 + 需要让 `super` 指向“定义当前方法的类的父类” → `super` 不能在运行时动态查找，必须在**类定义被求值时**就绑定到正确的父类上。Resolver 为每个有父类的子类创建**隐式作用域**，在其中定义 `super` 变量。
3. `super` 指向父类 + 需要同时访问 `this` → `super.method()` 相当于“在父类中找 `method`，但绑定到当前实例（`this`）上执行”。环境链中 `super` 和 `this` 被存储在相邻的环境中。
4. 不能自继承 + 需要静态检测非法 `super` 使用 → Resolver 检测三种非法 `super`：① 类外部使用 `super`；② 无父类的类内部使用 `super`；③ `class A < A` 自继承。

### 🧩 关键概念定义

| 概念 | 定义 | 对应代码/实现 |
| :--- | :--- | :--- |
| **继承（Inheritance）** | 子类复用父类的方法，通过 `class Child < Parent` 语法声明。 | `LoxClass.superclass` 字段 |
| **方法委派（Method Delegation）** | 在子类实例上调用方法时，若子类无该方法，则沿继承链向上查找。 | `LoxClass.findMethod()` 递归查找 |
| **`super` 静态绑定** | `super` 指向**定义当前方法的类的直接父类**，与运行时 `this` 的具体类型无关。 | Resolver 为 `super` 创建隐式作用域 |
| **动态字段（Dynamic Fields）** | Lox 字段不参与继承，只存在于实例自身的 `Map` 中。 | `LoxInstance.fields` 不沿继承链查找 |
| **隐式作用域（Implicit Scope）** | Resolver 为 `this` 和 `super` 主动压入的、不直接对应 AST 代码块的作用域。 | `visitClassStmt` 中的 `push` 操作 |
| **构造链（Constructor Chain）** | 子类 `init` 必须显式调用 `super.init()` 以确保父类字段被初始化。 | Lox 不强制，是一种工程契约 |

### ⚙️ 核心实践模块

```
LoxClass.findMethod():
  if (methods.containsKey(name)) return methods.get(name);
  if (superclass != null) return superclass.findMethod(name);
  return null;  // 找不到

Resolver.visitClassStmt:
  if (stmt.superclass != null) {
    resolve(stmt.superclass);  // 解析父类表达式
    beginScope();
    scopes.peek().put("super", true);  // 压入 super 隐式作用域
  }
  beginScope();
  scopes.peek().put("this", true);     // 压入 this 隐式作用域
  for (Stmt.Function method : stmt.methods) {
    resolveFunction(method, stmt.superclass != null && method.name.lexeme.equals("init"));
  }
  endScope();  // 弹出 this
  if (stmt.superclass != null) endScope();  // 弹出 super

Interpreter.visitSuperExpr:
  LoxClass superclass = (LoxClass) environment.getAt(distance, "super");
  LoxInstance instance = (LoxInstance) environment.getAt(distance - 1, "this");
  LoxFunction method = superclass.findMethod(expr.method.lexeme);
  if (method == null) throw new RuntimeError("Undefined property '" + expr.method.lexeme + "'.");
  return method.bind(instance);  // 方法绑定到当前实例上执行
```

### 🔥 常见认知陷阱与边界

1. **字段不参与继承**：Lox 字段不沿继承链查找，只存在于实例自身的 `Map` 中。父类字段和子类字段在同一个 `Map` 中，冲突时后者覆盖前者。这与 Java（继承链上每个类的字段都有独立存储）有本质区别。
2. **`super` 跳级调用**：`super` 只能跳一级（指向直接父类），不能写 `super.super.method()`。这是为了避免绕过中间层（`B` 的 `init` 或字段覆盖逻辑），导致状态不一致。
3. **构造链断裂**：子类 `init` 若不调用 `super.init()`，父类字段不会被初始化。Lox 不强制调用，因为它是动态语言，字段可以随时添加——但这种“自由”也带来了“半初始化对象”的风险。
4. **`super` 的静态性在“动态语言”中的特殊性**：Lox 虽然是动态语言，但 `super` 的语义是静态的，不依赖运行时类表。这与 Python 的 `super()`（动态 MRO 查找）有本质区别。


## 第十一章：系统全局概念索引

### 📊 语言层面的核心概念

| 概念 | 定义 | 关键章节 |
| :--- | :--- | :--- |
| **Lox** | 本书实现的脚本语言，具有动态类型、词法作用域、闭包、类和单继承特性。 | 全篇 |
| **动态类型（Dynamic Typing）** | 变量类型在运行时确定，无需编译期类型声明。 | 全篇 |
| **词法作用域（Lexical Scoping）** | 变量的可见范围由源代码中花括号的物理位置决定。 | 第 8 章 |
| **闭包（Closure）** | 函数对象持有其“诞生时”的环境引用，可以访问被捕获的自由变量。 | 第 7 章 |
| **一等函数（First-class Functions）** | 函数可以作为值传递、存储、返回。 | 第 7 章 |
| **类（Class）** | 用户自定义的复合数据类型，包含方法表。 | 第 9 章 |
| **单继承（Single Inheritance）** | 一个类只能有一个直接父类。 | 第 10 章 |
| **REPL（Read-Eval-Print Loop）** | 交互式执行环境，读取输入、求值、打印结果、循环。 | 第 1 章 |

### 🏗️ 编译器/解释器层面的核心概念

| 概念 | 定义 | 关键章节 |
| :--- | :--- | :--- |
| **扫描器（Scanner）** | 将字符流转换为 Token 流。 | 第 1 章 |
| **解析器（Parser）** | 将 Token 流转换为 AST（抽象语法树）。 | 第 3 章 |
| **AST（Abstract Syntax Tree）** | 抽象语法树，表达式的树形结构表示。 | 第 2 章 |
| **访问者模式（Visitor Pattern）** | 将操作与数据结构解耦的设计模式。 | 第 2 章 |
| **解释器（Interpreter）** | 遍历 AST 并执行求值。 | 第 4 章 |
| **环境（Environment）** | 变量名到值的映射表，带 `enclosing` 链。 | 第 5 章 |
| **Resolver（解析器/绑定器）** | 静态变量解析，计算变量距离。 | 第 8 章 |
| **表达式问题（Expression Problem）** | 扩展数据 vs 扩展操作的对立。 | 第 2 章 |
| **语法糖（Syntactic Sugar）** | 方便用户编写但不增加核心语言功能的语法结构（如 `for` 脱糖为 `while`）。 | 第 6 章 |

### 🔧 底层机制与数据流

| 概念 | 定义 | 关键章节 |
| :--- | :--- | :--- |
| **Token** | 源代码的最小语义单元。 | 第 1 章 |
| **Lexeme** | Token 对应的原始字符子串。 | 第 1 章 |
| **最长匹配（Maximal Munch）** | 词法分析中，能匹配更长的 Lexeme 就优先匹配更长的。 | 第 1 章 |
| **Pratt 解析器** | 优先级攀升算法，用一个带参数的循环处理表达式优先级。 | 第 3 章 |
| **后序遍历（Post-order Traversal）** | 先求值子节点，再求值父节点。 | 第 4 章 |
| **短路求值（Short-circuiting）** | `and`/`or` 根据左值决定是否求值右操作数。 | 第 4 章 |
| **环境链（Environment Chain）** | `Environment` 的 `enclosing` 链表，用于变量查找。 | 第 5 章 |
| **调用栈（Call Stack）** | JVM 的方法调用栈，用于执行流控制。 | 第 7 章 |
| **距离（Distance）** | 变量所在环境到当前环境的 `enclosing` 层数（静态计算的常量）。 | 第 8 章 |
| **绑定方法（Bound Method）** | 捕获了 `this` 实例的函数对象。 | 第 9 章 |
| **`super` 静态绑定** | `super` 指向定义当前方法的类的直接父类，与运行时 `this` 无关。 | 第 10 章 |
| **动态字段（Dynamic Fields）** | Lox 字段不继承，只存在于实例自身的 `Map` 中。 | 第 9、10 章 |
| **脱糖（Desugaring）** | 在解析阶段将语法糖（如 `for`）转换为更基础的结构。 | 第 6 章 |

### 🧩 设计模式与架构

| 概念 | 定义 | 应用章节 |
| :--- | :--- | :--- |
| **访问者模式（Visitor Pattern）** | 分离操作与数据结构，支持新操作的扩展。 | 第 2、4、7、8、9、10 章 |
| **策略模式（Strategy Pattern）** | 将算法封装为独立策略类（你提出的 `instanceof` 替代方案）。 | 第 4 章（讨论） |
| **工厂模式（Factory Pattern）** | 通过 `init` 劫持返回值实现（Lox 特有）。 | 第 9 章 |
| **单一职责原则（SRP）** | Parser 只负责解析，Resolver 只负责变量解析，Interpreter 只负责求值。 | 全篇 |
| **开闭原则（OCP）** | 对扩展开放（新增 Visitor），对修改关闭（不修改已有节点类）。 | 第 2 章 |
| **代码生成（Code Generation）** | 用元编程工具（`DefineAst.java`）生成样板代码，防止手写遗漏。 | 第 2 章 |

### 🔥 工程权衡与常见陷阱速查

| 陷阱/权衡 | 描述 | 应对方案 |
| :--- | :--- | :--- |
| **扫描器“按空格分割”谬误** | 扫描器不看空格，看字符类别（字母→标识符，数字→数值）。 | 基于字符类别的贪婪循环 |
| **字符串报错行号** | 未终止的字符串应指向首行，而非末行。 | 记录 `startLine` |
| **AST 丢弃括号** | 括号用于指导解析，构建完成后即失效。 | 树结构隐含优先级信息 |
| **访问者模式新增节点类的代价** | 新增 `Ternary` 需改所有 Visitor 实现。 | 代码生成 + 接受编译期报错 |
| **Pratt 右结合实现** | 右结合靠递归传参时**不加 1**。 | `parseExpression(precedence)`（不加 1） |
| **短路求值打破后序遍历** | `and`/`or` 是控制流结构，不是函数。 | `visitLogicalExpr` 中 `if` 判断 + 提前返回 |
| **`nil` 多语境规则** | `nil` 在算术中非法，在逻辑中合法（视为假）。 | 明确的语言契约 |
| **隐式全局声明的隐患** | `a = 5;` 自动创建全局变量，掩盖拼写错误。 | 建议 `var` 显式声明 |
| **`if(false)` 中变量声明** | `Resolver` 不报错，运行时才报 `Undefined variable`。 | Resolver 只负责词法可见性 |
| **字段遮蔽方法** | `instance.field = "x"` 覆盖同名方法。 | 属性查找顺序：先字段，后方法 |
| **`init` 返回值劫持** | `init` 可 `return something;` 丢弃实例。 | 工厂模式可行，但需用户明确 |
| **`super` 静态绑定 vs 动态 `this`** | `super` 指向定义类父类，`this` 指向运行时实例。 | Resolver 固定距离 + 方法绑定到 `this` |
| **字段不参与继承** | Lox 字段是动态 Map，不沿继承链查找。 | 与 Java 的静态字段布局区分 |


### 🚀 终章：从 jlox 到 clox 的桥梁

你现在已经完整掌握了 **jlox（树遍历解释器）**的全部核心知识：

| 维度 | 掌握程度 |
| :--- | :--- |
| **扫描器** | ✅ 完整实现，理解 Token、Lexeme、最长匹配、错误恢复 |
| **解析器** | ✅ 完整实现，理解 Pratt 解析、优先级、结合性 |
| **语义分析** | ✅ 完整实现，理解 Resolver、距离计算、词法作用域 |
| **解释器** | ✅ 完整实现，理解后序遍历、短路求值、类型检查 |
| **环境与状态** | ✅ 完整实现，理解环境链、变量生命周期、块级作用域 |
| **控制流** | ✅ 完整实现，理解分支、循环、语法糖脱糖 |
| **函数与闭包** | ✅ 完整实现，理解调用栈、环境链、异常机制、一等函数 |
| **面向对象** | ✅ 完整实现，理解类、实例、继承、`super` 静态绑定 |

### 🎯 下一步：字节码虚拟机（clox）

jlox 的优雅之处在于代码清晰，但代价是速度——每一次 AST 遍历都是 Java 对象树的递归跳转，无法充分利用现代 CPU 的流水线和缓存。

**clox（字节码虚拟机）**将把整个架构重写为 **C 语言**，核心变革是：
- 将 AST 编译为**紧凑的线性字节码指令序列**
- 用 **栈（Stack）** 替代 AST 递归求值
- 所有操作在 **字节码循环（Bytecode Loop）** 中执行
- 更接近真实硬件（如 Lua VM、Java VM、Python VM 的底层）

你的 jlox 知识体系将是学习 clox 的**最强预备**——你已深知“为什么要这么做”，现在将看到“如何做得更快”。

**准备好了吗？clox 的大门已经为你打开。**

