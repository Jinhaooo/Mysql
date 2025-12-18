# 什么是索引？
索引的定义就是帮助存储引擎快速找到数据的一种数据结构，也可以说是数据的目录。  

# 索引的分类
## 按数据结构分类
从数据结构的角度来看，MySQL 常见索引有 B+Tree 索引、HASH 索引、Full-Text 索引。  
<img width="811" height="421" alt="image" src="https://github.com/user-attachments/assets/75f9993a-5f62-4327-bda9-9abb182aa0fc" />  
InnoDB 是在 MySQL 5.5 之后成为默认的 MySQL 存储引擎，B+Tree 索引类型也是 MySQL 存储引擎采用最多的索引类型。  
在创建表时，InnoDB 存储引擎会根据不同的场景选择不同的列作为索引：  
如果有主键，默认会使用主键作为聚簇索引的索引键（key）；  
如果没有主键，就选择第一个不包含 NULL 值的唯一列作为聚簇索引的索引键（key）；  
在上面两个都没有的情况下，InnoDB 将自动生成一个隐式自增 id 列作为聚簇索引的索引键（key）；  
其它索引都属于辅助索引（Secondary Index），也被称为二级索引或非聚簇索引。创建的主键索引和二级索引默认使用的是 B+Tree 索引。  

