### 包含，而不是相等

理解 `term` 和 `terms` 是_包含_操作，而不是_相等_操作，这点非常重要。这意味着什么？

假如你有一个 term 过滤器 `{ "term" : { "tags" : "search" } }`，它将匹配下面两个文档：

```json
{ "tags" : ["search"] }
{ "tags" : ["search", "open_source"] } <1>
```

<1> 虽然这个文档除了 `search` 还有其他短语，它还是被返回了

回顾一下 `term` 过滤器是怎么工作的：它检查倒排索引中所有具有短语的文档，然后组成一个字节集。在我们简单的示例中，我们有下面的倒排索引：

| Token        | DocIDs  |
| ------------ | ------- |
|`open_source` | `2`     |
|`search`      | `1`,`2` |

当执行 `term` 过滤器来查询 `search` 时，它直接在倒排索引中匹配值并找出相关的 ID。如你所见，文档 1 和文档 2 都包含 `search`，所以他们都作为结果集返回。

提示：
  倒排索引的特性让完全匹配一个字段变得非常困难。你将如何确定一个文档_只能_包含你请求的短语？你将在索引中找出这个短语，解出所有相关文档 ID，然后扫描 _索引中每一行_来确定文档是否包含其他值。

由此可见，这将变得非常低效和开销巨大。因此，`term` 和 `terms` 是 _必须包含_ 操作，而不是 _必须相等_。

#### 完全匹配

假如你真的需要完全匹配这种行为，最好是通过添加另一个字段来实现。在这个字段中，你索引原字段包含值的个数。引用上面的两个文档，我们现在包含一个字段来记录标签的个数：

```json
{ "tags" : ["search"], "tag_count" : 1 }
{ "tags" : ["search", "open_source"], "tag_count" : 2 }
```

<!-- SENSE: 080_Structured_Search/20_Exact.json -->

一旦你索引了标签个数，你可以构造一个 `bool` 过滤器来限制短语个数：

```json
GET /my_index/my_type/_search
{
    "query": {
        "filtered" : {
            "filter" : {
                 "bool" : {
                    "must" : [
                        { "term" : { "tags" : "search" } }, <1>
                        { "term" : { "tag_count" : 1 } } <2>
                    ]
                }
            }
        }
    }
}
```

<!-- SENSE: 080_Structured_Search/20_Exact.json -->

<1> 找出所有包含 `search` 短语的文档
<2> 但是确保文档只有一个标签

这将匹配只有一个 `search` 标签的文档，而不是匹配所有包含了 `search` 标签的文档。