---
description: 'PROJECT #3 - QUERY EXECUTION'
---

# Query Execution

> This project mainly adds support for execution plans in the database system. We need to implement multiple **executors**, pass **query plans** into those executors, and run them. The following executors need to be implemented:
>
> * **Access Methods:** Sequential Scans, Index Scans
> * **Modifications:** Inserts, Updates, Deletes
> * **Miscellaneous:** Nested Loop Joins, Index Nested Loop Joins, Aggregation, Limit/Offset

We need to implement the iterator query processing model. Each query plan executor implements an `Init()` function and a `Next()` function. When the DBMS calls an executor's `Next()` function, it returns:

* a `tuple` and `true`, or
* `false` when there is no more `tuple` to return.

### TASK #1 - SYSTEM CATALOG

> This task is not difficult. It mainly requires implementing the functions specified in `src/include/catalog/catalog.h`. The catalog maintains database metadata, and these functions are related to database tables and indexes.

* `GetTable()` uses the `at` function of `std::unordered_map`. It performs bounds checking and throws an exception when the `key` does not exist.
* In `CreateIndex`, when creating an index for a table, use the table heap iterator to retrieve each tuple, then use the tuple's `KeyFromTuple` to construct the **key tuple** and insert it into the B+ tree.

***

### TASK #2 - EXECUTORS

#### SEQUENTIAL SCAN

> For the sequential scan executor, how should the iterator of `table_heap_` be stored? I did not find a clean solution at first, so I chose to use `std::vector<Tuple>` to store the results, then return them one by one in `Next`. When returning a result, remember to construct the returned tuple according to `OutputSchema`.

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

> Through the B+ tree index, first obtain the `RID`, then use the `RID` to retrieve the corresponding tuple from `table_heap_`.

* Use `dynamic_cast<BPlusTreeIndex<GenericKey<8>, RID, GenericComparator<8>> *>(indexInfo_->index_.get());` to cast `Index` to `BPlusTreeIndex`.

#### INSERT

> 1. The insert executor needs to distinguish whether the data to be inserted is `RawInsert` or comes from `child_executor_`.
> 2. If the target table has indexes, use `KeyFromTuple` to construct the `index_key`, then insert it into the B+ tree index.
> 3. For insert, update, and delete, the engine does not need to add tuples to `result_set`; otherwise the tests will fail because `result_set` is not empty.

#### UPDATE

> `child_executor_->Next` provides the tuple. Then call `GenerateUpdatedTuple` to generate the tuple to be updated, and finally use `table_heap_->UpdateTuple` to perform the update.

#### DELETE

> 1. `child_executor_->Next` provides the tuple.
> 2. Call `table_heap_->MarkDelete` to mark this tuple for deletion.
> 3. If the target table has indexes, use `KeyFromTuple` to construct the `index_key`, then delete this key from the B+ tree index.

#### NESTED LOOP JOIN

> Use tuples provided by `left_executor` and `right_executor`, then call `EvaluateJoin` to construct tuples that satisfy the join condition.

#### INDEX NESTED LOOP JOIN

> Use an index to perform the join, so the executor does not need to scan the entire inner table. Therefore, convert the outer tuple into the corresponding `key`, then look it up in the inner table index.

***

### Testing / Verification / Packaging

* Tests

```bash
cd build
make executor_test
```

* Format checks

```bash
make format
make check-lint
make check-clang-tidy
```

* Packaging

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

Then go to [**Gradescope**](https://www.gradescope.com/) and submit the code.
