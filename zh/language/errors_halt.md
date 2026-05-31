# XCX 3.1 错误处理（Halt）

XCX 使用结构化的 `halt` 系统来管理运行时状态与错误。

## Halt 级别

| 级别          | 行为                                           | 适用场景               |
|---------------|------------------------------------------------|------------------------|
| `halt.alert`  | 打印消息；执行继续。                           | 日志、非关键警告       |
| `halt.error`  | 输出至 stderr；中止当前帧。                    | 可恢复的逻辑错误       |
| `halt.fatal`  | 打印消息；立即终止 VM。                        | 严重故障、安全违规     |

## 示例

```xcx
halt.alert >! "Cache missed, fetching from DB...";

if (divisor == 0) then;
    halt.error >! "Division by zero!";
    return 0; --- Execution returns to caller from the recovery point
end;

if (NOT db.is_healthy()) then;
    halt.fatal >! "Database corrupted. Emergency shutdown.";
end;
```

## 语义错误与运行时 Panic

某些无效操作会触发自动 **panic**（根据上下文相当于 `halt.fatal` 或 `halt.error`）：

- **除零**（算术运算）
- **JSON 解析失败**：`json.parse()` 中无效字符串会导致 VM 立即退出。
- **路径遍历**：在 `store` 方法中使用 `..`，或使用项目根目录外的绝对路径，会触发 `halt.fatal`。
- **递归深度**：超过 800 帧会触发 `halt.error`。
