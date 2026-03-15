# Relation Database Management Systems (RDBMS)
A relational database like SQL is a collection of data items organized in tables.
ACID is a set of properties of relational database transactions.
* Atomicity - Each transaction is all or nothing
* Consistency - Any transaction will bring the database from one valid state to another
* Isolation - Executing transactions concurrently has the same results as if the transactions were executed serially
* Durability - Once a transaction has been committed, it will remain so

> Note: There are many techniques to scale a relational database: master-slave replication, master-master replication, federation, sharding, denormalization, and SQL tuning.

## Master-slave replication & Master-master replication
### [master-slave](./availability.md#master-slave)

### [master-master](./availability.md#master-master)

### [disadvantages](./availability.md#disadvantages)

## Federation
![federation](https://github.com/donnemartin/system-design-primer/raw/master/images/U3qV33e.png "")

* Federation (or functional partitioning) splits up databases by function.
* This results in less read and write traffic to each database
* This also reduces the datasize and replication lag
* Smaller datasets can be held in memory cache which increases the throughput and reduces the latency

### Disadvantages
* Not effective if schema requires huge functions or tables
* Not useful if includes a lot of joins (Requires server links)
* More hardware are complexity
