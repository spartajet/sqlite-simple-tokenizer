# rusqlite 迁移到 sqlx 的推荐方案

## 目标

- 降低 `rusqlite` 在业务代码中的使用范围；
- 在不破坏现有 FTS5 tokenizer 能力的前提下引入 `sqlx`；
- 控制迁移风险，保证每一步都可回滚。

## 现状与约束

当前项目并不是普通 SQLite CRUD 场景，而是要注册 FTS5 tokenizer。核心代码位于 `rusqlite-ext/lib.rs`，依赖了这些能力：

- `Connection::handle()` 获取底层 sqlite3 指针；
- `sqlite3_prepare_v3` / `sqlite3_bind_pointer` / `sqlite3_step` 等直接 FFI 调用；
- `Connection::extension_init2()` 用于扩展初始化流程。

`sqlx` 目前不直接提供上述等价能力，因此**无法直接替换** tokenizer 注册层。

## 推荐迁移路径（最小风险）

### Phase 1：分层（先做）

保持 tokenizer 注册层继续使用 `rusqlite-ext`，同时把“查询/写入”这类业务数据库访问抽象到单独模块，避免在业务逻辑中直接绑定具体库。

验收标准：

- tokenizer 注册行为与当前一致；
- 对外 API 不变。

### Phase 2：业务侧引入 sqlx（逐步）

对不依赖底层 sqlite3 指针的路径（如普通查询、批量写入）逐步迁移到 `sqlx`。建议优先迁移测试或独立命令路径，观察稳定性后再扩大范围。

验收标准：

- 与原有路径结果一致（查询结果、错误处理）；
- 迁移路径有回退开关或可快速回滚。

### Phase 3：长期评估（可选）

如未来 `sqlx` 提供完整的 tokenizer/extension 注册支持，或项目决定自维护更底层 SQLite FFI 层，再评估是否完全移除 `rusqlite`。

验收标准：

- FTS5 tokenizer 注册能力完全可替代；
- 不引入额外未覆盖的 unsafe 风险。

## 为什么这是“比较好的方案”

- **风险可控**：不触碰当前最敏感的 FFI 注册路径；
- **收益可见**：业务代码可以先获得 `sqlx` 的体验与生态；
- **可持续演进**：后续根据 `sqlx` 能力变化决定是否彻底替换，而不是一次性重写。
