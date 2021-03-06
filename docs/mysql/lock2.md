## 共享锁 / 排他锁 
共享锁/排他锁是innodb标准的行级锁

- 多个事务可以拿到一把共享S锁，共享锁之间不互斥，读读可以并行；  
- 而只有一个事务可以拿到排他X锁，排他锁与任何锁互斥，写写/读写不可以并行；

共享/排它锁的潜在问题是，不能充分的并行，解决思路是数据多版本

## 意向锁(Intention Locks)
意向锁是指，事务未来可能要加共享/排它锁了，先提前声明一个意向。
 
#### 1. 首先，意向锁，是一个表级别的锁(table-level locking)；  
#### 2. 意向锁分为：  
- 意向共享锁(intention shared lock, IS)，它预示着，事务有意向对表中的某些行加共享S锁  
- 意向排它锁(intention exclusive lock, IX)，它预示着，事务有意向对表中的某些行加排它X锁
 
举个例子：
```sql
select ... lock in share mode;  -- 要设置IS锁 
select ... for update;          -- 要设置IX锁 
```  
 
#### 3. 意向锁协议(intention locking protocol)并不复杂：  
- 事务要获得某些行的S锁，必须先获得表的IS锁  
- 事务要获得某些行的X锁，必须先获得表的IX锁  
 
#### 4. 由于意向锁仅仅表明意向，它其实是比较弱的锁，意向锁之间相互兼容，可以并行
 
#### 5. 虽然意向锁之间都相互兼容，但与共享锁/排它锁互斥，其兼容互斥表如下：

\ | S | X
  ---|---|---
IS | 兼容 | 互斥
IX | 互斥 |互斥

画外音：排它锁是很强的锁，不与其他类型的锁兼容。这也很好理解，修改和删除某一行的时候，必须获得强锁，禁止这一行上的其他并发，以保障数据的一致性。
 
## 插入意向锁(Insert Intention Locks)
针对数据的插入，插入意向锁
 
多个事务，在同一个索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。

## 思路总结
1. InnoDB使用共享锁，可以提高读读并发；
2. 为了保证数据强一致，InnoDB使用强互斥锁，保证同一行记录修改与删除的串行性；
3. InnoDB使用插入意向锁，可以提高插入并发； 