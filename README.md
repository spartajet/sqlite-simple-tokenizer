# sqlite-simple-tokenizer

![Crates.io License](https://img.shields.io/crates/l/sqlite-simple-tokenizer)

这里有一系列给 SQLite FTS5 编写的分词器！

## rusqlite -> sqlx 迁移方案

项目当前的 FTS5 tokenizer 注册逻辑依赖 `rusqlite::Connection` 暴露的底层 SQLite 句柄与 FFI 能力，
可以参考 [`docs/sqlx-migration-plan.md`](docs/sqlx-migration-plan.md) 中的分阶段方案。

结论：短期内保留 `rusqlite-ext` 作为 tokenizer 注册层，逐步把业务侧查询迁移到 `sqlx`，避免一次性重写导致风险过大。

## 许可

* Apache License, Version 2.0, ([LICENSE-APACHE](../LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
* MIT license ([LICENSE-MIT](../LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

### 贡献

除非您另有明确说明，否则任何您提交的代码许可应按上述 Apache 和 MIT 双重许可，并没有任何附加条款或条件。
