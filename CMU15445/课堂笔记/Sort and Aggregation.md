```toc
```
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407200846507.png)
我们假设磁盘大小远远大于内存
那么查询计划在执行的过程中，可能产生中间结果，而因为内存的不足，这个中间结果也需要被存储在磁盘上
## External Merge Sort
为什么需要排序？
- 关系型数据库本身无序
- 某些查询计划虽然没有显式指明order by，但是依然需要order by，如limit，distinct，group-by
数据能全部放到内存中，使用标准的排序算法即可（快排，归并）
外部排序，数据不在内存中，而在硬盘上

单趟排序被称为sorted run
用来排序的数据为k-v，假设我们需要order by colA，则colA为key，根据物化早晚我们可以得到两种value
1. 完整的tuple，提前物化
2. RID，延迟物化
选择提前物化，若tuple很长，那么排序的开销将会极大
选择延迟物化，排序的开销小，但是拿到RID后，还需要回表
我们应该根据tuple长度选择物化方式
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407200856527.png)

二路归并排序
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407200909610.png)
假设内存中只有三个frame
- 将每个page读入内存，内部排序后，写回磁盘（现在内存中每个page有两份）
- 两两归并

随着不断地归并，归并的两个对象越来越大，内存无法读入这么大的中间结果怎么办？
每次只将两个对象的第一个page读入内存，哪个对象的page归并完了就读取该对象的下一个page
三个frame，两个input存对象的page，一个ouput存归并结果

![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407200923158.png)
将二路归并扩展为n路归并
假设buffer pool有5个frame，那么内部排序后，得到的每个对象都有5个page，最后一个对象只有3个。一共22个对象
进行四四归并，需要进行22/4上取整次
此时得到的每个对象有20个page，最后一个对象只有8个page。一共6个对象
继续四四归并，得到2个对象，80个page和28个page
最后对这两个对象进行归并即可

也就是根据frame大小，确定对象的初始大小，最后将这个初始大小不断/B - 1，B为buffer pool的frame数量
## Aggregations
- Sorting
- Hashing
### Sorting Aggregate
当查询语句中具有order by子句时，我们通常需要使用Sorting Aggregate
### Hashing Aggregate
查询语句没有order by子句时，我们不需要使用sorting
**external hashing aggregate**
将需要group-by的属性作为key，假设中间结果非常大，我们需要将hash表分区落在硬盘上
去重：以需要去重的列值为key，将hash桶落到磁盘上，由于可能存在hash碰撞，将磁盘上的hash桶读回内存进行rehash。根据最后得到的hash table去重
聚合函数：以相应的聚合中间结果为value
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407201001340.png)
