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

## sqlx 如何操作 SQLite 数据库文件？与 rusqlite 是否一致？

### sqlx 操作 SQLite 文件的方式

`sqlx` 通过 SQLite 连接字符串（URL）访问数据库文件，常见形式：

- `sqlite::memory:`：内存库；
- `sqlite://data.db`：当前目录下文件；
- `sqlite:///absolute/path/to/data.db`：绝对路径文件。

通常会通过连接池（`SqlitePool`）或单连接（`SqliteConnection`）执行 SQL。对“普通建表、增删改查、事务”场景，和 `rusqlite` 在能力上基本一致。

### 与 rusqlite 的一致点

- 都是操作同一个 SQLite 引擎与同一种数据库文件；
- 都支持常见 SQL 与事务语义；
- 都可用于内存库和文件库。

### 与 rusqlite 的关键差异（本项目最相关）

- `rusqlite` 更偏底层，便于直接调用 SQLite C API / FFI；
- `sqlx` 更偏高层异步访问，默认不暴露本项目当前所需的底层句柄能力；
- 因此在本项目里：**业务查询层可迁移到 sqlx，FTS5 tokenizer 注册层仍需保留 rusqlite-ext**。

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
