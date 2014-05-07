##go-linq的并发模型
1. linq的处理被分为不同的步骤，步骤类型包括：

   * select （映射）

   * where （过滤）

   * from （设置数据源）

   * join （连接）

   * Group or distinct（分组）

   * 聚合 （折叠)

   * Set (集合的交集、差集、并集、串联）

   * 排序

   * 获取子分区


2. 每个步骤内部采取数据并行的方式进行处理

4. 每个步骤内的数据并行不采用每个数据一个GO routiner的简单模式（如果数据太多是不可行的），有2种不同的模式:

   * 如果数据是长度已知的列表，则采用数据分割，即分割成数量为（计算机内核数）的块，每块一个GO router

   * 如果数据长度未知（比如CHAN），则采用数据分块，即将接受的数据汇集为指定块的大小后，发给GO router处理，每个步骤的go router的数量=（计算机内核数）

3. 步骤之间尽可能不采用串联模式，而是尽量使用管道模式

   * select可以采取管道模式或分区模式（根据数据源的模式），但计划总以管道模式发送结果数据
   * where：总是结果数据分块后以管道模式发送给下个步骤
           这里有个问题，如果数据是列表采取数据分割的模式，则各分区的结果数可能非常不均衡，如果直接发送到下个步骤，将导致下个步骤各区的数据不均衡，各go routiner的负载不均衡。解决办法：数据不要直接按内核数分割，而是采取类似于处理channel的模式，将数据分割为包含较少数据的分块，以channel模式处理
   * from：分块可以管道，分区无需管道
   * join： 结果的数量无法预知，将结果数据分块后以管道模式发送给下个步骤
   * group: 结果必须在整个步骤完成后才能得到，因此可以采取分区模式发送给下个步骤
   * distinct: 结果以分区模式发给下个步骤
   * 聚合： 返回单个值，没有后续处理
   * Set： 除了串联其他都要在整个步骤完成后以分区模式发送，串联可以分块发送
   * 获取子分区： 数量已知的分区，其他情况分块
   * 排序，与group类似，结果也必须在整个步骤完成后得到，并且采用单routiner模式

4. 输出结果支持一次输出所有和提供CHAN两种模式。 如果单个元素处理结果

4. 如果数据数较少最好不进行并行处理，否则分割数据和汇总的开销会超过它们带来的好处

4. 数据并行处理模式总结：
   1. 数据源可以为list或者channel
   2. 每个数据源都可以转为list或channel，可以采取并行或者单线程模式，这个过程类似于MapReduce中的Map


## 尚未实现的linq查询运算符:

* 量词/Quantifiers：
Contains
All, Any, AnyWith, 
SequenceEqual

* 生成器方法
Range， Repeat

* 元素运算符/Element Operators
Single, 

## TODO list:
1. Add default func for OrderBy, DistinctBy...
2. Add keySelector func for Max, Min
3. Merge same operations if be before
4. Add channel mode for concat
5. Cancel other futures when error appears
6. Add trace for all futures?
7. 

