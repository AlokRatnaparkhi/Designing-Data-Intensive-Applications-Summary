# Designing Data-Intensive-Applications Summary
## Chapter 7 Transactions

## Weak Isolation Levels

### 1. Read Committed (No dirty reads and no dirty writes)
  - Implemented in most of the databases
  - Implemented in PostgreSQL, SQL server 2012, Oracle and MemSQL as default setting
  - Dirty writes are prevented using locks
  - Dirty reads are prevented by database by remembering both old commited value and value set by current transaction which is not yet committed. When other     
    transaction asks for read, old value is presented till other transaction does not commit new value.

### 1. Snapshot Isolation aka Repeatable read / (serialzable  in oracle)
Main problem - READ SKEW / NON REPEATABLE READ
- Non-repeatable read(fuzzy read) is that a transaction reads the same row at least twice but the same row's data is different between the 1st and 2nd reads because other transactions update the same row's data and commit at the same time(concurrently).
- Read skew is that with two different queries, a transaction reads inconsistent data because between the 1st and 2nd queries, other transactions insert, update or delete data and commit. Finally, an inconsistent result is produced by the inconsistent data.
  In summary, non-repeatable read is specifically about the inconsistency of a single read operation (reading same object twice) when the data is modified by another transaction between consecutive reads. On the other hand, read skew is a broader concept or generalization of NRR that involves inconsistencies in multiple reads (same object rows or different rows) within a transaction, especially when the transaction is making decisions based on those reads.

  Solution?

  Snapshot isolation - each transaction reads from a consistent snapshot of database

   Used in - backups and analytics
  - prevent dirty writes using locks (one write locks the object so that no other writes are performed at same time)
  - reader never blocks writer, writer never blocks readers
  - tag data which is written to DB using always increasing transaction ID
  - Use this tagging/snapshot to determine which reads to be executed (never read from row which has transaction ID more than the transaction which is currently running)

  ** Visibility rules:
  While transaction is getting executed
  - ignore writes from all other running or future transaction using tagged transaction id
  - ignore writes from any other aborted transactions
  - Make all writes visible to all application queries

## PROBLEMS WITH SNAPSHOT ISOLATION
1. Lost Updates
  
Lost updates occur when two transaction read - modify -write data (cycle) concurrently. It may happen that second transaction which finishes last does not include modification done by first transaction and overwrites it. 

How to prevent?

- Atomic Operations:
    
    - Most of the databases support atomic operation that takes up a lock while reading and updating the object and does not release the lock until the object is updated. Any subsequent transactions will have to wait until first transaction finishes its operations. Example of atomic operation,
   UPDATE USERS SET USERNAME = 'Batman' WHERE USER_ID = 10  (This is considered as atomic operation by database. It will lock the row number 10 till update is not completed hence preventing any subsequent statments to read the value.
   - Also, known as Cursor stability

- Explicit Locking:

   Sometimes single atomic operation is not possible cause we have to implement multiple statements as part of transactions. So, explicitly lock the rows which you want to modify using SQL statement as supported by databases.
   E.g. BEGIN TRANSACTION;
     SELECT * FROM USERS WHERE COUNTRY = 'INDIA' FOR UPDATE; -- For Update keyword is used for explicitly locking the rows
     UPDATE USERS SET STATE = 'MH' WHERE CITY IN ('AURANGABAD', 'PUNE', 'KOLKATA', 'HYDERABAD')
   COMMIT;
   - Downside ? -> Explicit locking can lead to race condition
     
- Automatic Lost Update Detection
   
    - Let transaction manager of database automatically detects the lost updates. Keep retrying the transaction which are lost updates.
    - But how?. Can be easily identified using snapshot isolation (we know that which data is stale using snapshots / versions of row)
    - Implemented in Postgres and MS SQL to detect lost updates
    -  Does not require application code, SQL statement or explicit locking so less error prone
    
- Compare and Set
   
  - Only allow update if the value is not changed after your last read. If it is modified then retry.
  - can be easily implemented using SQL statements e.g.
  UPDATE WIKI SET CONTENT = 'NEW CONTENT' WHERE CONTENT = ' OLD CONTENT' --the where part checks whether value has not changed since last read
  - Less reliable. Check how database vendor implements this compare ans set operation.

2.  Write Skew
   - Generalization of lost update problem
   - two transactions read from same objects at the same time, based on the information provided by the reads, they update same or different objects. If it is same object then you can get dirty write or lost update anomoly. But, if objects are different then it may lead to write skew anomoly.
   - Famous example, 2 doctors must be in hospital for a shift. Among three doctors, two update the shift record at the same time. At the time of read both read that 2 doctors are available at hospital. Both update their respective status objects to unavailable based on information from first query but clearly system should not allow it.
- Can not be solved using atomic operations, automatic detection
- Can be solved by either using Serializable isolation level or using explicit lock (FOR UPDATE while firing select queries)

## Serializable isolation level & 2PL
- Serializable isolation level guarantees that even multiple transactions are executed concurrently, they will be executed as if they are executed serially by the database.

- It is the strictest of all isolation levels.

##How to implement?
- Literally execute transactions serially. (E.g., executing all transactions using single execution thread)
pros : No concurrency problem
cons: Low throughput, worst performance
- Using 2PL (two phase locking) — most popular
- Optimistic concurrency control techniques (Serializable snapshot isolation)

##Two Phase Locking
1. There are two kinds of locks for all transactions
    - Shared locks
    - Exclusive locks
2. Locks are acquired at the beginning of transaction and released at the end of transactions. Due to these two phases, it is called as two phase locking (2PL).

3. Both readers and writers acquire locks. Writers block both readers and writers. Readers block the writers.

4. If transaction wants to read, it acquire shared lock if there is no other transaction hold exclusive lock on it. Multiple readers can share reader locks. Writer can not acquire exclusive lock on object until readers finishes the read (i.e., as long as there are any shared or exclusive lock on object).

6. If transaction wants to write to object, it checks whether there are any shared or exclusive on object, if not it acquires it. Other transactions (both readers and writers) should wait until this transaction releases a lock on object.

pros:
  - Ensure serializability
  protects from concurrency issues like write skey/ phantoms which are not possible by other isolation levels ( Using Predicate locks & Index Range locks)
  - Ensure better read performance than executing all transactions serially

cons:
  - still, poor performance but better than first soltion
  - low throughput
  - too many locks
  - Is susceptible to deadlock due to locks
