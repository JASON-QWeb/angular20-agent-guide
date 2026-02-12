# PostgreSQL 数据库设计与优化指南 (Agent 规则版)

本指南为 AI Agent 提供 PostgreSQL 数据库设计规范、查询优化及生产环境最佳实践。

---

## 1. 命名与结构规范 (Naming & Conventions)

### 1.1 命名约定
- **表名/字段名**：统一使用 `snake_case` (下划线命名)，且使用**小写**。
- **表名单复数**：推荐使用复数（如 `users`, `orders`）。
- **主键**：统一命名为 `id`，类型推荐使用 `BIGSERIAL` 或 `UUID` (v7)。

### 1.2 数据类型选择
- **字符串**：优先使用 `text` 而非 `varchar(n)`，除非有硬性长度约束。
- **时间**：**必须**使用 `timestamptz` (带时区的时间戳)。
- **JSON**：涉及非结构化数据或扩展属性时，**必须**使用 `jsonb`。

---

## 2. 索引优化 (Indexing)

### 2.1 索引原则
1.  **外键索引**：所有 `FOREIGN KEY` 必须手动创建索引（PostgreSQL 不会自动创建）。
2.  **复合索引**：遵循“最左前缀原则”。将过滤性最强的列放在最前面。
3.  **覆盖索引**：利用 `INCLUDE` 子句减少回表。
4.  **JSONB 索引**：
    - 全文搜索或复杂查询使用 `GIN` 索引。
    ```sql
    CREATE INDEX idx_data_gin ON my_table USING GIN (data_jsonb);
    ```

### 2.2 避免索引失效
- 避免在索引列上使用函数：`WHERE date_trunc('day', created_at) = ...` (会导致索引失效)。
- 修复方式：使用范围查询 `WHERE created_at >= ... AND created_at < ...`。

---

## 3. 高级 SQL 技巧

### 3.1 批量操作
使用 `INSERT INTO ... ON CONFLICT (id) DO UPDATE SET ...` 实现 Upsert。

### 3.2 窗口函数 (Window Functions)
在处理排名、分页总数或统计时，优先使用窗口函数（如 `ROW_NUMBER()`, `RANK()`, `COUNT(*) OVER()`）。

### 3.3 CTE (Common Table Expressions)
使用 `WITH` 提高复杂查询的可读性。对于性能敏感的查询，注意 `MATERIALIZED` 的使用。

---

## 4. 后端集成 (以 Go 为例)

1.  **驱动选择**：优先使用 `jackc/pgx/v5`，它比 `lib/pq` 更现代且性能更好。
2.  **连接池**：**必须**配置连接池参数（MaxConns, MinConns, MaxConnIdleTime）。
3.  **迁移工具**：推荐使用 `golang-migrate/migrate` 或 `pressly/goose`。

---

## 5. 性能诊断

当 Agent 遇到查询慢的问题时，应执行：
1.  **EXPLAIN ANALYZE**：查看执行计划，确认是否走了索引。
2.  **统计信息**：`ANALYZE table_name` 更新采样信息。
3.  **查询日志**：检查 `pg_stat_statements` 插件记录的高耗时查询。

---

## 6. Agent 指令 (Agent Instructions)

当你在进行数据库建模或查询编写时，请遵循：
1. **语义化设计**：表结构必须清晰表达业务实体。
2. **性能预判**：在大表查询时必须考虑索引，严禁编写可能导致全表扫描的代码。
3. **安全第一**：严禁拼接 SQL 字符串，必须使用参数化查询防止 SQL 注入。
4. **一致性**：根据业务需求合理选择事务隔离级别。
