1. 并行联接算法：使用多线程联接两个关系表
   - **hash join**
   
   - **sort-merge join**

   - nested-loop join
   
     - Simple Nested-Loops Join:
   
       ```
       for each row r in R
       	for each row s in S
       		if r和s 满足联接条件
               	output tuple<r, s>
       ```
   		优化器一般会选择 **联接列含有索引**的表作为内部表
     	
   	由于 **内部表扫描次数总是索引的高度**，所以当 R和S联接列都有索引且索引高度相同时，**选择记录少的表作为外部表**
     	
     - Block Nested-Loops Join: 针对没有索引的联接情况，使用join buffer来减少内部循环读取表的次数
     
       
   
2. 许多面向OLTP的DBMS没有实现hash联接，小规模结果的**带索引嵌套循环联接(index nested-loop join)**基本与hash联接 等价
3. 联接算法的目标：
   - 最小化同步，避免上(闩)锁
   - 最小化CPU缓存miss，使工作线程满足数据的局部性
     - Cache 和 TLB(快表)的性能
     - 时间与空间的局部性

4. hash join 是 **OLAP**型DBMS 中**最重要**的操作，要尽可能利用多核来加速；分为三个阶段(假设R 联接 S)：
   - Partition (可选)：对用于联接的key(字段的值)使用hash，将R和S的元组分为不同的集合
     - 可以减少第二阶段Build时的cache miss。理想情况下，partition的代价 比 cache miss的代价更小
     - 非阻塞式划分法：(1) 所有线程一起更新一个**全局的划分集**，需要上锁来同步 (2) 每个线程拥有**自己的划分集**，在所有线程完成后，要把这些划分集合并在一起
     - 阻塞式划分法(radix hash join)：需多次扫描产生划分
   - Build：扫描关系R(元组 或 划分)，根据联接的key建立*哈希表T*
     - hash函数——速度 与 碰撞 的trade-off
       - CRC-32 (1975)
       - MurmurHash (2018)
       - Google CityHash (2011)
       - CLHash (2016)
     - hash模式——空间大的哈希表 与 碰撞后额外的查找指令 的trade-off
       - 开放链式：哈希表的每个槽中 维护一个链表
       - 线性探测：碰撞时，线性地查找下个空位(需要表的空间比较大)
       - Robin Hood：（线性探测的变式）每个Key会记录自己离最优位置的距离(比如key1 本来应该在 [2] 的位置，现在保存在[4]，距离就为2)；当插入key2时，若key2在 [4]的距离为3，**大于**key1的距离2，则**key2会取代key1的位置**，key1往后挪(*劫富济贫*)
       - Cuckoo：使用多个不同的hash函数对应多个hash表。在插入时，随便挑选一个有空槽的表，若都没有空槽，从这些表中驱逐出一个元素，重新hash找到新位置——若进入无限循环，需要重建表；查找的时间复杂度永远是O(1)
   - Probe：对于S的每个元组*x*，在*哈希表T*中查找*x*的联接key值。若找到匹配的，则输出联接的元组
     - 如果R和S经过划分，则可以给每个线程分配一个独立的partition
     - 当key很有可能不在hash表中，可使用**Bloom Filter(布隆过滤器)**来加速——如果检测结果为是，该元素**不一定**在集合中；但如果检测结果为否，该元素**一定不在**集合中