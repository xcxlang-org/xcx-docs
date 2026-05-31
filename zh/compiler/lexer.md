
# XCX 词法分析器（Scanner）— v3.1

词法分析器负责将原始源字节流转换为离散的 token 流。

## 实现细节

- **文件**：`src/lexer/scanner.rs`
- **技术**：在 `&[u8]` 上进行手动、逐字节扫描。
- **API**：单一方法 `next_token(&mut self, interner: &mut Interner) -> Token`，由解析器按需调用（非迭代器）。
- **前瞻**：通过 `peek()` 实现单字节前瞻，通过 `peek_next()` / `peek_at(offset)` 实现双字节前瞻。

## 内部结构

```rust
pub struct Scanner<'a> {
    source:   &'a [u8],   // borrowed reference to original source — no allocation
    pos:      usize,      // byte position
    char_pos: usize,      // Unicode character position (for Span.col)
    line:     usize,
    col:      usize,
}
```

`Scanner` 将源字符串借用为字节切片。不会预先转换为 `Vec<char>`——所有扫描均在字节级别进行。`char_pos` 仅对非 UTF-8 续字节（`10xxxxxx`）递增，因此 Unicode 字符计数对 Span 报告保持正确，而无需解码每个字符。

## Token 类型

Token 在 `src/lexer/token.rs` 中定义为 `TokenKind` enum。每个 `Token` 携带 `Span { line, col, len }` 用于错误报告。`len` 以 Unicode 字符（通过 `char_pos` 差值）计量，而非字节。

主要类别：

| 类别 | 示例 |
|---|---|
| Literals | `IntLiteral(i64)`, `FloatLiteral(f64)`, `StringLiteral(StringId)`, `True`, `False` |
| Type keywords | `TypeI`, `TypeF`, `TypeS`, `TypeB`, `Array`, `Set`, `Map`, `Table`, `Json`, `Date`, `Fiber` |
| Set type keywords | `TypeSetN`, `TypeSetQ`, `TypeSetZ`, `TypeSetS`, `TypeSetB`, `TypeSetC` |
| Control flow | `If`, `Then`, `ElseIf`, `Else`, `End`, `While`, `Do`, `For`, `In`, `To`, `Break`, `Continue` |
| Functions/Fibers | `Func`, `Return`, `Fiber`, `Yield` |
| Operators | `Plus`, `Minus`, `Star`, `Slash`, `Caret`, `PlusPlus`, `Has`, `And`, `Or`, `Not` |
| Set operators | `Union`, `Intersection`, `Difference`, `SymDifference` |
| Special punctuation | `GreaterBang` (`>!`), `GreaterQuestion` (`>?`), `DoubleColon` (`::`), `DoubleComma` (`,,`), `Bridge` (`<->`) |
| Builtins | `Net`, `Serve`, `Store`, `Halt`, `Terminal`, `Json`, `Date` |
| Special | `RawBlock(StringId)`, `AtStep`, `AtAuto`, `AtWait` |

## 特殊扫描特性

### Raw Blocks
由 `<<<` 与 `>>>` 界定。中间所有内容捕获为单个 `RawBlock(StringId)` token，用于内联 JSON 或多行字符串数据。

```
<<<
  { "key": "value" }
>>>
```

检测使用 `self.source[self.pos..].starts_with(b"<<<")`——在主 match 分支的 `<` 消歧中打开。原始内容逐字节捕获，直到找到 `>>>`。

### Comments
XCX 使用 `---` 作为注释分隔符。检测使用 `self.source[self.pos..].starts_with(b"---")`：
- **单行**：`--- this is a comment`（`---` 后同一行有非空白内容）
- **多行**：`---` 后至行尾仅含空白则打开块注释，由 `*---` 关闭

多行关闭检测使用 `self.source[self.pos..].starts_with(b"*---")`。扫描器在消费 `---` 后向前查看以决定适用哪种模式。

### Unicode Set Operators
扫描器通过 `starts_with` 识别 Unicode 符号的 UTF-8 字节序列：
- `∪` → `TokenKind::Union`
- `∩` → `TokenKind::Intersection`
- `\` (ASCII backslash) → `TokenKind::Difference`
- `⊕` → `TokenKind::SymDifference`

对于多字节 Unicode 运算符，在 leading byte（`c >= 128`）之后调用适当次数的 `advance()` 以消费剩余续字节。

### Else/ElseIf Disambiguation
扫描器在识别 `else` / `els` 后向前查看，检查下一个词是否为 `if`——若是，则将两个词合并为单个 `ElseIf` token。保存的位置（`after_ws_pos`、`after_ws_char_pos`、`after_ws_line`、`after_ws_col`）允许扫描器在下一个词不是 `if` 时回退。

### `@` Directives
以 `@` 开头的 token 通过消费 ASCII 字母字节并匹配结果来扫描：
- `@step` → `AtStep`
- `@auto` → `AtAuto`
- `@wait` → `AtWait`

未知的 `@` 序列产生 `TokenKind::Unknown('@')`。

### Dot-Dot (`..`) → `To`
两个连续点 `..` 扫描为 `To` token（用于范围表达式），与单个 `.`（`Dot`）不同。

### Double-Comma (`,,`) → `DoubleComma`
两个连续逗号 `,,` 扫描为 `DoubleComma`（用于集合范围字面量：`set:N { 1,,10 }`）。

### Identifier Scanning
`identifier()` 捕获 ASCII 字母数字字节、下划线及 `>= 128` 字节的连续序列（以包含 UTF-8 标识符）。捕获的字节范围通过 `std::str::from_utf8` 转换并小写化以匹配关键字。先检查大小写敏感匹配（例如集合类型的 `"N"`、`"Q"`、`"Z"`，大写关键字变体的 `"UNION"`、`"HAS"`、`"AND"`）。非关键字标识符传递给 `Interner::intern()`。

### Number Scanning
`number()` 累积 ASCII 数字字节。若遇到 `.` 后跟数字，token 变为 `FloatLiteral`。字节范围通过 `str::from_utf8` 转换并用 `.parse()` 解析。

### String Scanning
`string()` 逐字节处理转义序列（`\n`、`\t`、`\r`、`\"`、`\\`、八进制 `\NNN`、十六进制 `\xHH`），构建 `Vec<u8>` 后通过 `String::from_utf8` 转换。结果进行内联。

## 字符串分配

所有标识符与字符串字面量通过 `Interner::intern()` 处理，返回 `StringId (u32)`。原始 `String` 在内联器内部 `Vec<String>` 中只存储一次；流水线其余部分使用数字 ID，消除类型检查与编译期间的堆比较。
