```toc
```
## Cost Estimation
代价的估计需要基于统计信息，大部分关系型数据库都维护了统计信息
当然，我们也能手动更新统计信息
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161938570.png)

引入一些概念
$N_{R}$：R表的tuple数量
$V(A, R)$：对R表的attributie A去重后，唯一值的数量
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161941742.png)

选择基数$SC(A, R)$，根据attribute A筛选R表，平均能筛出来的记录数量
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202407161944425.png)
使用SC时，我们假设attribute数据均匀分布，即出现频率相同


## Plan Enumeration