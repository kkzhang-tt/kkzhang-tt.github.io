---

layout: post
title: "密集索引 VS 稀疏索引"
author: "kkzhang"
category: "database"
image: 5.jpg
---

对于数据库索引来说，其中一个性质是索引可以是密集的，也可以是稀疏的。关于索引类型的选择，需要根据数据库的特性进行权衡（trade-off）。

假设存在以下数据库记录：

<img src="https://raw.githubusercontent.com/kkzhang-tt/kkzhang-tt.github.io/main/_images/index_1.png" alt="index_1" style="zoom: 67%;" />

该 table 存在四列，并且表中的行记录被分为四页，每页包含四条记录。我们指定 first_name 字段作为索引。

### 密集索引

<img src="https://raw.githubusercontent.com/kkzhang-tt/kkzhang-tt.github.io/main/_images/index_2.png" alt="index_2" style="zoom: 33%;" />

从上图我们可以看到，对于密集索引来说，对于表中的每一个 first_name 行记录都会存在一个索引项。如果我们想要查询 first_name 为 Rachelle 的用户，我们需要在索引上执行二分查找，确定该行的位置。

### 稀疏索引

<img src="https://raw.githubusercontent.com/kkzhang-tt/kkzhang-tt.github.io/main/_images/index_3.png" alt="index_3" style="zoom:33%;" />

与密集索引相反，一个稀疏索引会对应多条行记录。

从上图看到，我们只用了四个稀疏索引条目就完成对所有表行记录的映射（每个索引对应一个页）。这种情况下，如果我们想要查询 first_name 为 Rachelle 的用户，我们可以在稀疏索引上同样执行二分查找（但是不同于密集索引，二分查找只能确定目标项在索引中的边界范围），我们发现 Rachelle 处于 Loyd 与 Shepherd 之间。当找到边界之后，我们需要在以 Loyd 开始的页中进行扫描查找 Rachelle 所在的行（该扫描过程可以使用二分查找进行优化）。

我们注意到，上述表中的记录极硬处于有序状态，这是稀疏索引的一个限制：稀疏索引要求数据必须有序，不然第二步中的扫描就不能找到目标数据。

### 总结

在数据更新的时候，密集索引需要更多的维护，因为每个数据库行都会有一个索引，当数据库记录被插入，更新，删除时，都需要维护相应的索引项。而且，由于每个行记录都需要维护一个索引项，意味着密集索引需要更多的内存空间。不过密集索引的优势也比较明显，目标记录能够通过一次二分查找就能很快定位，并且密集索引不需要强制数据有序存储。

对于稀疏索引来说，数据更新时，需要较小的维护负担，因为每个索引都包含多条记录（数据记录的子集）。这种轻量级的维护负担意味着插入，更新，删除操作能够更快执行。同时，索引项的数目较少意味着更小的内存空间。但是对于数据查询会比较慢，因为需要两次查找才能定位目标数据（一次是查找稀疏索引，一次是在索引维护的数据块中进行查找）。稀疏索引只有在有序数据的情况下才能工作。
