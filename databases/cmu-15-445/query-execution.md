---
description: 'PROJECT #3 - QUERY EXECUTION'
---

# 😉 Query Execution

> 这个 project 主要是实现增加对数据库系统的执行计划的支持。需要实现各种 **executors**，将 **query plan** 传入 executors 然后执行它们。需要实现下列的 executors：
>
> * **Access Methods:** Sequential Scans, Index Scans
> * **Modifications:** Inserts, Updates, Deletes
> * **Miscellaneous:** Nested Loop Joins, Index Nested Loop Joins, Aggregation, Limit/Offset

我们需要实现迭代查询处理模型，每个查询计划执行器实现了一个 `Init()` 函数和 `Next()` 函数。当 DBMS 调用 executors 的 `Next()` 函数时，它会返回

* 一个 `tuple` 并返回 `true`
* 没有 `tuple` 可以再返回时，返回 `false`

### TASK #1 - SYSTEM CATALOG

> 这个 task 不难，主要是实现 `src/include/catalog/catalog.h` 文件（catalog 维护了数据库的 meta-data）中的要求实现的函数，这些函数与数据库的表和索引有关。

* `GetTable()` 使用 `std::unordered_map` 的 `at` 函数，它会做下标检查，当 `key` 不存在时会抛出异常
* 在 `CreateIndex` 函数中，创建表的索引时，需要使用 table heap 的迭代器取出每个 tuple，然后使用 tuple 的 `KeyFromTuple` 构造 **key tuple** 插入到 B+ Tree 中

***

### TASK #2 - EXECUTORS

#### SEQUENTIAL SCAN

> 在顺序执行器中，如何保存 `table_heap_` 的迭代器呢？一直没找到解决方案，因为我先选择在使用 `std::vector<Tuple>` 先保存结果，然后在 `Next` 中一个一个返回。在返回结果时，记得要根据 `OutputSchema` 构造 tuple 返回

| <pre class="language-cpp" data-line-numbers><code class="lang-cpp">Tuple make_tuple(const Tuple &#x26;tuple, const Schema *output_schema) {
  std::vector&#x3C;Value> values;
  for (const auto &#x26;col : output_schema->GetColumns()) {
    values.push_back(tuple.GetValue(output_schema, output_schema->GetColIdx(col.GetName())));
  }
  return Tuple(values, output_schema);
}
</code></pre> |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### INDEX SCANS

> 通过 B+ Tree 索引，先获得 `RID`，然后根据 `RID` 在 `table_heap_` 中得到对应的 tuple

* 通过 `dynamic_cast<BPlusTreeIndex<GenericKey<8>, RID, GenericComparator<8>> *>(indexInfo_->index_.get());` 将 `Index` 转成 `BPlusTreeIndex` 类型

#### INSERT

> 1. 插入执行器需要区分待插入的数据是 `RawInsert` 还是来自 `child_executor_`
> 2. 如果待插入的表存在索引需要使用 `KeyFromTuple` 构造 `index_key`，然后将它插入到 B+ Tree 索引中
> 3. engine 在插入、更新、删除不需要将 tuple 添加到 `result_set` 中，否则在 test 中会报 `result_set` 不为空的错误

#### UPDATE

> 由 `child_executor_` 的 `Next` 提供 tuple，然后调用 `GenerateUpdatedTuple` 生成待更新的 tuple，最后使用 `table_heap_->UpdateTuple` 进行更新操作

#### DELETE

> 1. 由 `child_executor_` 的 `Next` 提供 tuple
> 2. 调用 `table_heap_->MarkDelete` 标记这个 tuple 需要删除
> 3. 如果待插入的表存在索引需要使用 `KeyFromTuple` 构造 `index_key`，然后在 B+ Tree 索引中将这个 Key 删除

#### NESTED LOOP JOIN

> 使用 `left_executor` 和 `right_executor` 提供的 tuple 进行 `EvaluateJoin` 构造符合条件的 tuple

#### INDEX NESTED LOOP JOIN

> 使用索引来进行 Join，这样就不需要扫描整个 inner table。因此我们需要将 outer tuple 转化为对应的 `key`，然后在 inner table index 中进行查找。

***

### 测试/验证/打包

* 测试

```bash
cd build
make executor_test
```

* 格式验证

```bash
make format
make check-lint
make check-clang-tidy
```

* 打包

```bash
zip project3-submission.zip src/include/buffer/lru_replacer.h src/buffer/lru_replacer.cpp \
	src/include/buffer/buffer_pool_manager.h src/buffer/buffer_pool_manager.cpp \
	src/include/storage/page/b_plus_tree_page.h src/storage/page/b_plus_tree_page.cpp \
	src/include/storage/page/b_plus_tree_internal_page.h src/storage/page/b_plus_tree_internal_page.cpp \
	src/include/storage/page/b_plus_tree_leaf_page.h src/storage/page/b_plus_tree_leaf_page.cpp \
	src/include/storage/index/b_plus_tree.h src/storage/index/b_plus_tree.cpp \
	src/include/storage/index/index_iterator.h src/storage/index/index_iterator.cpp \
	src/include/catalog/catalog.h src/include/execution/execution_engine.h src/include/storage/index/index.h \
	src/include/execution/executor_factory.h src/execution/executor_factory.cpp \
	src/include/execution/executors/seq_scan_executor.h src/execution/seq_scan_executor.cpp \
	src/include/execution/executors/index_scan_executor.h src/execution/index_scan_executor.cpp \
	src/include/execution/executors/insert_executor.h src/execution/insert_executor.cpp \
	src/include/execution/executors/update_executor.h src/execution/update_executor.cpp \
	src/include/execution/executors/delete_executor.h src/execution/delete_executor.cpp \
	src/include/execution/executors/nested_loop_join_executor.h src/execution/nested_loop_join_executor.cpp \
	src/include/execution/executors/nested_index_join_executor.h src/execution/nested_index_join_executor.cpp \
	src/include/execution/executors/limit_executor.h src/execution/limit_executor.cpp \
	src/include/execution/executors/aggregation_executor.h src/execution/aggregation_executor.cpp \
	src/include/storage/index/b_plus_tree_index.h src/storage/index/b_plus_tree_index.cpp
```

然后前往 [**https://www.gradescope.com**](https://www.gradescope.com/) 提交代码
