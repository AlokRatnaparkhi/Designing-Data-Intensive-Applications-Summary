# Designing-Data-Intensive-Applications-Summary
## Chapter 7 Transactions

## Weak Isolation Levels

### 1. Read Committed (No dirty reads and no dirty writes)
  - Implemented in most of the databases
  - Implemented in PostgreSQL, SQL server 2012, Oracle and MemSQL as default setting
  - Dirty writes are prevented using locks
  - Dirty reads are prevented by database by remembering both old commited value and value set by current transaction which is not yet committed. When other     
    transaction asks for read, old value is presented till other transaction does not commit new value.
