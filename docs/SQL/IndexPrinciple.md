
<!-- TOC -->

- [1. 索引底层原理](#1-索引底层原理)
    - [1.1. InnoDB引擎的索引](#11-innodb引擎的索引)
        - [1.1.1. InnoDB索引实现](#111-innodb索引实现)
            - [1.1.1.1. InnoDB逻辑结构页介绍](#1111-innodb逻辑结构页介绍)
            - [1.1.1.2. 为什么InnoDB会使用B+树？](#1112-为什么innodb会使用b树)
            - [1.1.1.3. InnoDB索引B+tree实现过程](#1113-innodb索引btree实现过程)
                - [1.1.1.3.1. 页分裂/页合并](#11131-页分裂页合并)
            - [1.1.1.4. 联合索引（复合索引）的底层实现](#1114-联合索引复合索引的底层实现)
            - [1.1.1.5. B+树中一个节点到底多大合适？](#1115-b树中一个节点到底多大合适)
    - [1.2. MyISAM引擎的索引](#12-myisam引擎的索引)
    - [1.3. Hash索引介绍](#13-hash索引介绍)

<!-- /TOC -->

# 1. 索引底层原理  
&emsp; 不同的存储引擎支持的索引类型不一样：  

* InnoDB支持事务，支持行级别锁定，支持B-tree、Full-text等索引，不支持Hash索引；  
* MyISAM不支持事务，支持表级别锁定，支持B-tree、Full-text等索引，不支持Hash索引；  

## 1.1. InnoDB引擎的索引  
### 1.1.1. InnoDB索引实现
#### 1.1.1.1. InnoDB逻辑结构页介绍  
&emsp; 参考[MySql存储引擎](/docs/SQL/13.MySqlStorage.md)  

#### 1.1.1.2. 为什么InnoDB会使用B+树？  
<!--
https://mp.weixin.qq.com/s/jWIdb4PFSF9o6zRlBnFMQA
-->
* BTree  
    &emsp; BTree是平衡搜索多叉树，设树的度为2d（d>1），高度为h，那么BTree要满足以一下条件：  
    * 每个叶子结点的高度一样，等于h；
    * 每个非叶子结点由n-1个key和n个指针point组成，其中d<=n<=2d，key和point相互间隔，结点两端一定是key；
    * 叶子结点指针都为null；
    * 非叶子结点的key都是[key,data]二元组，其中key表示作为索引的键，data为键值所在行的数据；

    &emsp; BTree的结构如下：  
    ![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-91.png)  
    &emsp; 在BTree的机构下，就可以使用二分查找的查找方式，查找复杂度为h*log(n)，一般来说树的高度是很小的，一般为3左右，因此BTree是一个非常高效的查找结构。  

* B+Tree树  
    &emsp; B+Tree是BTree的一个变种，设d为树的度数，h为树的高度，B+Tree和BTree的不同主要在于：  

    * B+Tree中的非叶子结点不存储数据，只存储键值；
    * B+Tree的叶子结点没有指针，所有键值都会出现在叶子结点上，且key存储的键值对应data数据的物理地址；
    * <font color = "red">B+Tree的每个非叶子节点由n个键值key和n个指针point组成；</font>  
    
    &emsp; B+Tree的结构如下：  
    ![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-92.png)  

&emsp; <font color = "red">为什么索引结构默认使用B+Tree，而不是BTree，二叉树，红黑树？</font>  
1. 操作系统中以页这种结构作为读写的基本单位。  
2. 操作系统IO消耗：<font color = "red">一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。</font>这样的话，索引查找过程中就要产生磁盘I/O消耗，相对于内存存取，I/O存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数。  
3. 具体分析：（重点B树和B+树的对比）  
    1. 二叉树：树的高度不均匀，不能自平衡，查找效率跟数据有关（树的高度），并且IO代价高。  
    2. 红黑树：树的高度随着数据量增加而增加，IO代价高。 
    3. **<font color = "lime">B树：</font>**  
        1. B树中每个节点中不仅包含数据的key值，还有data值。而每一个页的存储空间是有限的，<font color = "lime">如果data数据较大时将会导致每个节点（即一个页）能存储的key的数量很小。**当存储的数据量很大时同样会导致B树的深度较大，**增大查询时的磁盘I/O次数进而影响查询效率。</font>  
        2. 范围查询，磁盘I/O高。示例：  
            ![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-93.png)  
            &emsp; 假设需要访问所有『大于 4，并且小于 9 的数据』：如果不考虑任何优化，在上面的简单 B 树中需要进行4次磁盘的随机 I/O 才能找到所有满足条件的数据行：
            1. 加载根节点所在的页，发现根节点的第一个元素是 6，大于 4；
            2. 通过根节点的指针加载左子节点所在的页，遍历页面中的数据，找到 5；
            3. 重新加载根节点所在的页，发现根节点不包含第二个元素；
            4. 通过根节点的指针加载右子节点所在的页，遍历页面中的数据，找到 7 和 8；  
            &emsp; 当然可以通过各种方式来对上述的过程进行优化，不过 B 树能做的优化 B+ 树基本都可以，所以不需要考虑优化 B 树而带来的收益。
    4. **<font color = "lime">B+树：</font>** 
        1. **<font color = "red">从物理存储结构上说，B-Tree和B+Tree都以页(4K)来划分节点的大小，但是由于B+Tree中中间节点不存储数据，因此B+Tree能够在同样大小的节点中，存储更多的key，提高查找效率。</font>**
        2. **<font color = "red">叶子节点之间会有个指针指向，这个也是B+树的核心点，可以大大提升范围查询效率，也方便遍历整个树。</font>** 
        3. **<font color = "red">B+tree的查询效率更加稳定。由于非终结点并不是最终指向文件内容的结点，而只是叶子结点中关键字的索引。所以任何关键字的查找必须走一条从根结点到叶子结点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。</font>**   

----
&emsp; B+Tree对比BTree的优点：  
1. 磁盘读写代价更低  
&emsp; 一般来说B+Tree比BTree更适合实现外存的索引结构，因为存储引擎的设计专家巧妙的利用了外存（磁盘）的存储结构，即磁盘的最小存储单位是扇区（sector），而操作系统的块（block）通常是整数倍的sector，操作系统以页（page）为单位管理内存，一页（page）通常默认为4K，数据库的页通常设置为操作系统页的整数倍，因此索引结构的节点被设计为一个页的大小，然后利用外存的“预读取”原则，每次读取的时候，把整个节点的数据读取到内存中，然后在内存中查找，已知内存的读取速度是外存读取I/O速度的几百倍，那么提升查找速度的关键就在于尽可能少的磁盘I/O，那么可以知道，每个节点中的key个数越多，那么树的高度越小，需要I/O的次数越少，因此一般来说B+Tree比BTree更快，因为B+Tree的非叶节点中不存储data，就可以存储更多的key。  
2. 查询速度更稳定  
&emsp; 由于B+Tree非叶子节点不存储数据（data），因此所有的数据都要查询至叶子节点，而叶子节点的高度都是相同的，因此所有数据的查询速度都是一样的。  

#### 1.1.1.3. InnoDB索引B+tree实现过程  
<!-- 
https://mp.weixin.qq.com/s/6BoGlaYpdDjzZy19YhInEw
https://zhuanlan.zhihu.com/p/98818611
-->
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-78.png)  
&emsp; InnoDB的B+tree的节点：  

* 叶子节点，存储行数据，叫做目录页。  
* 非叶子节点，存储索引数据，叫做页目录。  

1. 数据插入过程中，数据页中的记录是按主键正序排列的。实际上就是为了能够使用二分查找法快速定位一条记录。  
2. 一页插满会进行页分裂。  

##### 1.1.1.3.1. 页分裂/页合并  
<!-- 
https://mp.weixin.qq.com/s/6BoGlaYpdDjzZy19YhInEw
-->

#### 1.1.1.4. 联合索引（复合索引）的底层实现
&emsp; <font color = "red">联合索引（复合索引）的底层实现？最佳左前缀原则？</font>  
&emsp; 联合索引底层还是使用B+树索引，并且还是只有一棵树，只是此时的排序会：首先按照第一个索引排序，在第一个索引相同的情况下，再按第二个索引排序，依次类推。  
&emsp; 这也是为什么有“最佳左前缀原则”的原因，因为右边（后面）的索引都是在左边（前面）的索引排序的基础上进行排序的，如果没有左边的索引，单独看右边的索引，其实是无序的。  

#### 1.1.1.5. B+树中一个节点到底多大合适？  
<!-- 
InnoDB中一棵B+树可以存放多少行数据？
https://mp.weixin.qq.com/s/QKLX7zNm7xxMZ7dYvlkxxw
-->
&emsp; B+树中一个节点到底多大合适？  
&emsp; 1页或页的倍数最为合适。因为如果一个节点的大小小于1页，那么读取这个节点的时候其实也会读出1页，造成资源的浪费。所以为了不造成浪费，所以最后把一个节点的大小控制在1页、2页、3页等倍数页大小最为合适。  
&emsp; 在 MySQL中B+ 树的一个节点大小为“1页”，也就是16k。 

## 1.2. MyISAM引擎的索引  
&emsp; <font color = "red">MyISAM也是B+树结构，但是MyISAM索引的叶子节点的数据保存的是行数据的地址。</font>因此，MyISAM中索引检索的算法首先在索引树中找到行数据的地址，然后根据地址找到对应的行数据。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-37.png)  
&emsp; MyISAM的索引文件仅仅保存数据记录的地址。主键索引和辅助索引，只是主索引要求key是唯一的，而辅助索引的key可以重复。如果在Col2上建立一个辅助索引，则此索引的如下图：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-38.png)  

&emsp; <font color = "red">在MyISAM储存引擎中，数据和索引文件是分开储存的，Myisam的存储文件有三个，后缀名分别是 .frm、.MYD、MYI，其中 .frm 是表的定义文件，.MYD 是数据文件，.MYI 是索引文件。</font>  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-36.png)  

* .frm文件：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等。
* .MYD (MYData) 文件：MyISAM 存储引擎专用，用于存储MyISAM 表的数据。  
* .MYI (MYIndex)文件：MyISAM 存储引擎专用，用于存储MyISAM 表的索引相关信息。  

## 1.3. Hash索引介绍  
&emsp; 哈希索引底层的数据结构就是哈希表。对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码（hash code），并且Hash索引将所有的哈希码存储在索引中，同时在索引表中保存指向每个数据行的指针。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-39.png)  
&emsp; 在 MySQL 中，只有 Memory 存储引擎显式的支持哈希索引，而innodb是隐式支持哈希索引的。  

&emsp; <font color = "red">哈希索引适用的场景</font>：等值查询。  
&emsp; <font color = "red">哈希索引不适用的场景</font>：hash函数的不可预测性，hash索引中经过hash函数建立索引之后，索引的顺序与原顺序无法保持一致。不支持范围查询、不支持索引完成排序、不支持联合索引的最左前缀匹配规则、不支持部分匹配、只支持等值查询如=，IN()，不支持 < >。  