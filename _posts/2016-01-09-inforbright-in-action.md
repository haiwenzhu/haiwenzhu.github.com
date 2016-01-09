---
layout: post
title: "Inforbright实践"
categories:
  - Inforbright
  - Database
---

最近做的项目都是和数据查询展示相关的，而且对应的数据量都比较大，某些表每天的数据都在千万级、整体数据量亿级以上，这样的数据量放在MySQL有些吃不消，即使是分库分表查询效率也上不出，而且太复杂的分表或导致数据的查询逻辑也变得复杂，所以从MySQL换到了Infobright（以下简称IB）上，选择IB最大的一个好处是其查询语法和MySQL完全兼容，所以业务逻辑基本不需要修改。记录一下这段时间以来IB的使用心得。

### 数据导入
首先是数据导入，IB不支持insert、update、delete这样的通常的数据更新方式，只支持`LOAD DATA INFILE ... INTO`，换句话说，只能通过cvs格式的文件导入。IB数据导入的速度还是很快的，在导入数据的时候，IB会同时开启多个线程对多个不同的列同时做导入，这个是其内部实现，没办法外部控制。实测下来，一千万行、18列的数据，导入时间在1分20秒左右。  
我们自己采用的数据更新方式是先把数据更新到MySQL，再把数据从MySQL导出成文件，然后导入到IB，IB数据的组织是基于视图和不同的数据版本。更新分两种，增量更新和全量更新，我们的数据源通常是按天提供数据的，对于这种数据的更新采用的是增量更新，增量更新只需要把数据load到当前视图指向的表就行。如果涉及到历史数据的变更就需要全量更新IB的数据，当全量导入时，数据会被导入到一个新的表，导入完成之后，再将视图切换到新导入的这张表上。查询是从视图中查询数据，这样虽然存储数据的表名会变化，但是业务查询的视图名称不需要变化。  
数据先更新到MySQL的主要原因是为了应对数据的更新，尤其是数据的修改和删除。IB数据基于视图的好处是当全量导入数据的时候，数据的查询不会受到影响，否则需要将表drop掉（IB不支持`truncate table`语法）再重新创建表然后再导入数据(其实不用视图也可以用`RENAME TABLE`，不过IB不支持这个语法)。  
![数据导入流程](http://i.imgur.com/d33s7gb.png?1)


### 数据查询
官方文档列出的最有效的数据类型有TINYINT、 SMALLINT、 MEDIUMINT、 INT、 BIGINT、DECIMAL、DATE、 TIME ，其次是CHAR和VARCHAR，所以建表的时候优先考虑使用这些数据类型，有一点需要注意的是不支持UNSIGNED类型的数据。对于CHAR或者VARCHAR的数据，当数据列的取值范比较小，数据相对收敛的时候（官方推荐当`数据总数/唯一数据数`大于10时，就可以考虑使用lookups），可以考虑使用lookups特性，lookups类似于一个数据字典，IB会用整型替换掉相应的CHAR或者VARCHAR的数据，以达到提高查询效率和降低存储空间的目的。lookups的使用也很简单，在数据表定义的时候，在需要使用lookups的列加上`comment 'lookup'`注释就行。  
数据查询时，尽量避免使用OR，而是改用IN或者UNION ALL，在我们的应用中发现，UNION ALL的效果比IN好，`SELECT f1,f2 FROM table WHERE f1=1 UNION ALL SELECT f1,f2 FROM table WHERE f1=2`比`SELECT f1,f2 FROM table WHERE f1 IN (1,2)`的查询效率要高。  
还有一点就是尽量避免使用`SELECT *`，因为IB是列式存储的，每一列都可能涉及到数据的解压缩，所以最好只取需要的列。  
其他的一些MySQL的查询优化对IB也是适用的，比如使用LIMIT、查询条件里避免使用函数、数据取值尽量避免NULL等。  
说一下IB的查询效率，以我们自己的项目来说，在没有缓存的前提下，MySQL分钟级别的查询，IB基本都在秒级。查询效率提升在5-10倍左右。另外说一点就是IB对并发查询支持得不好，如果查询并发比较高的话需要注意。

### 内部实现
内部实现这部分，只是我自己对看过的一些东西的一些理解，试着写一下。
最最重要的一点，IB是一个列式数据库。列式数据库的优点有更高效的数据压缩和基于列的查询。IB所有的优点的前提也是基于数据的列式存储。   
IB内部数据的存储分成了三层：
- Data Packs(DPs)，每一列的数据分组存到DPs里，一个DPs最多存放65536个记录，数据压缩也是在DPs里进行压缩的
- Data Pack Nodes(DPNs)，和DPs是一对一的关系，保存了DPs里的数据的一些信息，包括最大值、最小值、加和等值
- Knowledge Nodes(KNs)，相比于DPNs来说，KNs保存了更全局的和DPNs以及列相关的数据，甚至和其他表的关联数据。DPNS除了在数据load的时候会生成之外，在查询时也能会变化，可以理解为是一份动态的数据。

IB在查询的时候，会根据DPNs和KNs的信息尽可能的过滤掉无关的DPs，以减少需要解压缩的DPs。比如某个DPs的值范围为0到10，如果我们查询的是该列小于0的数据，则这个DPs就可以忽略掉。而如果查询的是该列小于5的数据，则需要对这个DPs的数据做解压缩，以获得最终的数据。  
顺便提一下IB的数据压缩，文档里写的平均的数据压缩比和传统的行式数据库比起来在1：10，我们项目的实际应用下来发现和MySQL比起来，数据压缩也在10倍左右。

### 总结
通常数据查询优化的思路，一是减小查询的数据集，二是优化数据查询的效率。对于第一种思路，除了选择合理的表结构设计和字段类型，还有一种方法是离线数据汇总，把一些大的数据集做汇总，汇总成一些小的数据结果集，以达到减小查询数据集的目的。第二种思路通常是合理的sql和索引等，这里就不赘述。

抛砖引玉，欢迎拍砖交流。

### 参考连接
1. [Infobright Technology white paper](http://www.azinta.com/Services/EN-IB_Arch_IEE%20white_paper_Final.pdf)
2. [Infobirght wiki](https://www.infobright.org/index.php/ICE_Wiki/wiki-4/)

最后，增图一张(高清大图请点[这里](http://hpi.de/fileadmin/user_upload/fachgebiete/naumann/projekte/RDBMSGenealogy/RDBMS_Genealogy_V5.jpg))：
![RDBMS_Genealogy_V5](http://hpi.de/fileadmin/_processed_/csm_RDBMS_Genealogy_V5_dc41ea0c12.jpg)

