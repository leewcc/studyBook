# mc 的存储机制
mc 采用 slab allocator 管理内存，每个 slab 会被分割成多个 chunk，每个 slab 的 chunk 都是相等大小的，如 88 B、112B、144B... 等固定大小的 chunk。
 
内存分配的原则：
1. 当一个新的数据要被存放时，首先会根据数据大小选择合适的 slab（每个 slab 的chunk 大小都是相等的）
2. 查看当前 slab 是否有空闲的 chunk 可存放，如果有则直接存放进去，如果没有则申请一个新的 slab（page），再将其按 chunk 大小切割进行存储
3. 如果无法申请 slab，则会对刚刚分配的 slab 进行 LRU，而不是对整个 memcached 进行 LRU