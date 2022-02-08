# 1 MySQL Architecture

### Mysql Logical Archietcture&#x20;

![](<../../../.gitbook/assets/Screen Shot 2022-02-07 at 8.40.32 PM.png>)

### Concurrency Control

* the server level&#x20;
* storage-engine level.

#### Read/ Write Lock [https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

Read lock / Shared lock

Exclusive lock / Write lock

* One way to improve the concurrency of a shared resource is to be more selective about what you lock.
* Most commercial database servers, you get what is known as row-level locking in your tables, with a variety of often complex ways to give good performance with many locks.

#### Lock Strategies

Table locks

Table locks have variations for improved performance in specific situations. For example, READ LOCAL table locks allow some types of concurrent write operations. Write and read lock queues are separate with the write queue being wholly of higher priority than the read queue.

Row Locks

This strategy allows multiple people to edit different rows concurrently without blocking one another.

This enables the server to take more concurrent writes, but the cost is more overhead in having to keep track of who has each row lock, how long they have been open, and what kind of row locks they are as well as cleaning up locks when they are no longer needed.

Row locks are implemented in the storage engine, not the server.

### Transactions

A transaction is a group of SQL statements that are treated atomically, as a single unit of work.

It’s all or nothing.

```
1  START  TRANSACTION;
2  SELECT balance FROM checking WHERE customer_id = 10233276;
3  UPDATE checking SET balance = balance - 200.00 WHERE customer_id = 10233276;
4  UPDATE savings SET balance = balance + 200.00 WHERE customer_id = 10233276;
5  COMMIT;
```

ACID

Atomicity

Consistency

Isolation

Durability
