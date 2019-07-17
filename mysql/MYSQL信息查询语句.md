### MySQL内部信息查看语句

1. 查看每张表的每列的详细

   ```mysql
   show full columns form tablename;
   ```

2. 查看表的基本信息

   ```mysql
   desc tablename;
   ```

3. 查看事务提交模式

   1或者 ON表示启用    0或者OFF表示禁用

   ```mysql
   show variables like 'AUTOCOMMIT'
   set AUTOCOMMIT = 1;
   ```

4. 查看事务的隔离级别

   ```mysql
   --查看全部隔离级别
   select @@tx_isolation;
   --查看系统当前隔离级别
   select @@global.tx_isolation;
   
   --设置全局会话隔离级别
   set global TRANSACTION ISOLATION LEVEL REPEATABLE READ;
   --设置当前会话隔离级别
   set SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
   ```

5. 查看表的相关信息

   ```mysql
   show table status like 'user' \G
   ```

   

6. 

