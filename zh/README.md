# XCX 技术文档套件

> 📌 **请注意**：本中文文档由人工智能翻译生成，可能存在不准确之处或翻译错误。如需最精确的信息，请参阅[英文版文档](../en/README.md)。

欢迎阅读 XCX 3.1 与 XCX 编译器的官方技术文档。

## 📖 语言参考
关于 XCX 语言语法与功能的全面指南。

- [语法基础](language/syntax.md)：注释、标识符与代码块。
- [变量与常量](language/variables.md)：声明、重新赋值与遮蔽。
- [数据类型](language/types.md)：简单类型、复合类型与类型转换。
- [运算符](language/operators.md)：算术、字符串、逻辑、比较与集合。
- [控制流](language/control_flow.md)：条件语句与循环。
- [函数与纤程](language/functions_fibers.md)：子程序、协程与委托。
- [集合](language/collections.md)：数组、集合、映射与表（关系型）。
- [JSON 与 HTTP](language/json_http.md)：数据交换、网络与安全。
- [日期与时间](language/dates.md)：创建、字段与格式化。
- [I/O 与终端](language/io_terminal.md)：输入、输出、延迟与系统命令。
- [字符串方法](language/string_methods.md)：字符串工具完整列表。
- [错误处理](language/errors_halt.md)：停机级别（alert、error、fatal）。
- [标准库](language/library_modules.md)：内置模块（`crypto`、`store`、`env`）。

## ⚙️ 编译器内部机制
深入介绍 XCX 编译器与虚拟机的工作原理。

- [架构概览](compiler/architecture.md)：编译流水线。
- [词法分析器](compiler/lexer.md)：词法化与递归扫描。
- [语法分析器](compiler/parser.md)：Pratt 解析与 AST 生成。
- [语义分析](compiler/semantics.md)：类型检查与符号解析。
- [虚拟机](compiler/vm.md)：栈式机架构与操作码。

## 📦 工具
XCX 生态系统工具指南。

- [PAX 手册](pax/pax_manual.md)：包管理与项目结构。

---
*文档版本：3.0.0 *
