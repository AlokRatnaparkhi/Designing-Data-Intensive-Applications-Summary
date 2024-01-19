# Designing Data-Intensive-Applications Summary
## Chapter 7 Transactions

## Weak Isolation Levels

### 1. Read Committed (No dirty reads and no dirty writes)
  - Implemented in most of the databases
  - Implemented in PostgreSQL, SQL server 2012, Oracle and MemSQL as default setting
  - Dirty writes are prevented using locks
  - Dirty reads are prevented by database by remembering both old commited value and value set by current transaction which is not yet committed. When other     
    transaction asks for read, old value is presented till other transaction does not commit new value.

### 1. Snapshot Isolation



##Lost Updates
  
Lost updates occur whent two transaction read - modify -write (cycle) concurrently. It may happen that second transaction which finishes last does not include modification done by first transaction and overwrites it. 

How to prevent?

1. Atomic Operations - Most of the databases support atomic operation that takes up a lock while reading and updating the object and does not release the lock until the object is updated. Any subsequent transactions will have to wait until first transaction finishes its operations. Example of atomic operation,
   UPDATE USERS SET USERNAME = 'Batman' WHERE USER_ID = 10  (This is considered as atomic operation by database. It will lock the row number 10 till update is not completed hence preventing any subsequent statments to read the value.
   - Also, known as Cursor stability

2. Explicit Locking
   Sometimes single atomic operation is not possible cause we have to implement multiple statements as part of transactions. So, explicitly lock the rows which you want to modify using SQL statement as supported by databases.
   E.g. BEGIN TRANSACTION;
     SELECT * FROM USERS WHERE COUNTRY = 'INDIA' FOR UPDATE; -- For Update keyword is used for explicitly locking the rows
     UPDATE USERS SET STATE = 'MH' WHERE CITY IN ('AURANGABAD', 'PUNE', 'KOLKATA', 'HYDERABAD')
   COMMIT;
   - Downside ? -> Explicit locking can lead to race condition

3. Automatic Lost Update Detection
    - Let transaction manager of database automatically detects the lost updates. Keep retrying the transaction which are lost updates.
    - But how?. Can be easily identified using snapshot isolation (we know that which data is stale using snapshots / versions of row)
    - Implemented in Postgres and MS SQL to detect lost updates
    -  Does not require application code, SQL statement or explicit locking so less error prone
    
4. Compare and Set
  - Only allow update if the value is not changed after your last read. If it is modified then retry.
  - can be easily implemented using SQL statements e.g.
  UPDATE WIKI SET CONTENT = 'NEW CONTENT' WHERE CONTENT = ' OLD CONTENT' --the where part checks whether value has not changed since last read
  - Less reliable. Check how database vendor implements this compare ans set operation. 
