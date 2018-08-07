# Mysql的存储引擎

MySQL有多种存储引擎，每种存储引擎有各自的优缺点:   
MyISAM、InnoDB、MERGE、MEMORY(HEAP)、BDB(BerkeleyDB)、EXAMPLE、FEDERATED、ARCHIVE、CSV、BLACKHOLE。  
MyISAM与InnoDB是常用的两个存储引擎。  

## MyISAM & InnoDB 比较
- 每个MyISAM在磁盘上存储成三个文件。第一个文件的名字以表的名字开始，扩展名指出文件类型。.frm文件存储表定义。数据文件的扩展名为.MYD (MYData)。
- MyISAM表格可以被压缩，而且它们支持全文搜索。
- MyISAM不支持事务，不支持外键。如果事物回滚将造成不完全回滚，不具有原子性。
- MyISAM在进行updata时进行表锁，并发量相对较小。如果执行大量的SELECT，MyISAM是更好的选择。InnoDB是行锁，高并发方面优势巨大。
- MyISAM的索引和数据是分开的，并且索引是有压缩的，内存使用率就对应提高了不少。能加载更多索引，而Innodb是索引和数据是紧密捆绑的，没有使用压缩从而会造成Innodb比MyISAM体积庞大不小。
- MyISAM缓存在内存的是索引，不是数据。而InnoDB缓存在内存的是数据，相对来说，服务器内存越大，InnoDB发挥的优势越大。
- InnoDB中不保存表的行数，如select count(\*) from table时，InnoDB需要扫描一遍整个表来计算有多少行，但是MyISAM只要简单的读出保存好的行数即可。注意的是，当count(*)语句包含where条件时MyISAM也需要扫描整个表
- 对于自增长的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中可以和其他字段一起建立联合索引
- 清空整个表时，InnoDB是一行一行的删除，效率非常慢。MyISAM则会重建表
- InnoDB支持行锁（某些情况下还是锁整表，如 update table set a=1 where user like '%lee%';在where条件没有使用主键时，比如DELETE FROM mytable这样的删除语句。)

## MyISAM与InnoDB的优缺点
### MyISAM
优点：查询数据相对较快，适合大量的select，可以全文索引。   
缺点：不支持事务，不支持外键，并发量较小，不适合大量update

### InnoDB  
优点：支持事务，支持外键，并发量较大，适合大量update。   
缺点：查询数据相对较快，不适合大量的select

对于支持事物的InnoDB类型的表，影响速度的主要原因是AUTOCOMMIT默认设置是打开的，
而且程序没有显式调用BEGIN 开始事务，导致每插入一条都自动Commit，严重影响了速度。
可以在执行sql前调用begin，多条sql形成一个事物（即使autocommit打开也可以），将大大提高性能。

## MyISAM与InnoDB的选择使用
如果你的应用程序一定要使用事务，毫无疑问你要选择INNODB引擎。  
如果你的应用程序对查询性能要求较高，就要使用MYISAM了。MYISAM索引和数据是分开的，而且其索引是压缩的，可以更好地利用内存。
所以它的查询性能明显优于INNODB。压缩后的索引也能节约一些磁盘空间。  
MYISAM拥有全文索引的功能，这可以极大地优化LIKE查询的效率。  
现在一般都是选用innodb，主要是myisam的全表锁，读写串行问题，并发效率锁表，效率低。myisam对于读写密集型应用一般是不会去选用的。

## 提升InnoDB性能的方法：  
针对InnoDB来说，影响性能的主要是 innodb_flush_log_at_trx_commit 这个选项，如果设置为1的话，那么每次插入数据的时候都会自动提交，导致性能急剧下降，应该是跟刷新日志有关系，设置为0效率能够看到明显提升，当然，同 样你可以SQL中提交“SET AUTOCOMMIT = 0”来设置达到好的性能。另外，还听说通过设置innodb_buffer_pool_size能够提升InnoDB的性能，但是我测试发现没有特别明显 的提升。

基本上我们可以考虑使用InnoDB来替代我们的MyISAM引擎了，因为InnoDB自身很多良好的特点，比如事务支持、存储 过程、视图、行级锁定等等，在并发很多的情况下，相信InnoDB的表现肯定要比MyISAM强很多，当然，相应的在my.cnf中的配置也是比较关键 的，良好的配置，能够有效的加速你的应用。
任何一种表都不是万能的，只用恰当的针对业务类型来选择合适的表类型，才能最大的发挥MySQL的性能优势。
