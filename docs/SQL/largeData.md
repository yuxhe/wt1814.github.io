

# 大数据量操作  
## 在线修改表结构  

<!-- 

http://www.phppan.com/2012/05/online-schema-change/
在线修改MySQL大表的表结构
https://www.cnblogs.com/jamesbd/p/3581957.html

-->

&emsp; 在线修改MySQL大表的表结构：  
1. 方案1、直接ALTER TABLE  
&emsp; 这个方案只能说这仅仅是一种方案，在某些非实时在线或数据量较小时有较好的表现。  
2. 方案２、模拟数据库修改表结构的操作，在非数据库层实现整个过程。  
&emsp; 实现业务中对于数据的读写分离  
&emsp; 创建一个已经按需求修改好结构的新表  
&emsp; 修改业务逻辑，将读操作指向旧表，将写操作指向新表。如果读旧表没有，再读新表，并将旧的数据写入到新表，当然这一步写入操作我们可以不用，我们可以在后台做一个定时任务将旧数据同步到新表。  
&emsp; 这种方案有一个较大的缺点，需要业务逻辑层配合实现数据的迁移，对于业务逻辑有修改，并且如果有多台机器的话，需要一台一台的修改，较费时间，但是对于MySQL的两种主要存储引擎都适用。  
3. 方案３、facebook online schema change  
&emsp; facebook的OSC在整体流程上与方案2没有较大的区别，只是它在这里引入了触发器，从而不需要修改业务逻辑，在数据库层就实现了新数据的两个表的同步问题。其大概步骤如下：  
    1. 按需求创建新表  
    2. 针对原始表创建触发器  
    3. 对于原始表的更新操作都会被触发器更新到新表中  
    4. 把原始表中的数据复制到新表中  
    5. 将新表替换旧表  
    6. fb的osc方案从数据库层解决了方案2的问题，但是它仅支持InnoDB存储引擎。  
4. 方案４、换一个思路，保留字段。  
&emsp; 假设一切可以从头再来，可以加多一些冗余字段，各个类型都加一些，备用。只是，回不去了！  
5. 方案5、再换一个思路，增加扩展表。  
&emsp; 不在原有的表的基础上修改了，以增加扩展表的方式，将新字段的数据写入到扩展表中，修改业务逻辑，这些字段从新表中读取。  

### pt-online-schema-change  
<!-- 
在线修改大表结构pt-online-schema-change
https://segmentfault.com/a/1190000014924677
-->