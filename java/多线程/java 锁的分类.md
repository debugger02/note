https://blog.csdn.net/qq_30718137/article/details/106089642

https://www.cnblogs.com/xumBlog/p/11978997.html


## 锁的分类

乐观锁：假定没有冲突，在修改数据如果发现数据和之前获取的不一致，则读取最新的数据，再次尝试修改。
悲观锁：假定会发生冲突，同步所有的数据的相关操作，从读数据就开始上锁。

独占锁：给资源加上写锁，当前线程可以修改资源，其它线程不能再加锁（单写）
共享锁：给资源加上读锁后只能读不能改，不能加写锁（多读）

可重入锁：同时加两次锁，不会出现死锁（再次拿到锁）
不可重入锁：同时加两次锁，会出现死锁（阻塞）

公平锁：抢到锁的顺序和抢锁顺序相同则为公平锁
非公平锁：抢到锁的顺序和抢锁顺序无关。

**synchronized 的特性：**  

1. 用于实例方法、静态方法时，隐式指定锁对象
2. 用于代码块时，显示指定锁对象
3. 锁的作用域：对象锁，类锁，分布式锁
4. 是可重入锁，也是独享锁和悲观锁
5. 锁的状态：偏向锁/轻量级锁/重量级锁

