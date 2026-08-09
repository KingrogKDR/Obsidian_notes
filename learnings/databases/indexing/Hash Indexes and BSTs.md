Hash indexes are basically hashmaps that map a key to a value. This value may be anything that contains the actual value or at least addresses it.

- Hash indexes can never be hash sets because we only store a single value in hash sets and therefore, it is mainly used in situations where we just want to know if something exists or not.

Hash indexes provide O(1) for every read and write operation, which is it's main advantage.

However, hash indexes are bad on the disk as represented. In theory, it will still be O(1) operation, but in practice, it takes much longer than that. And this is a good example why if something is true in theory doesn't mean it should be true in practice.

![Why hash indexes aren't preferred on the disk](disk_no_hashmap.png)

Also, another disadvantage of hash indexes is range queries. 

For e.g let's say the user wants all the data for the people who spent between $100 and $200.

For this operation, the read time for hash indexes will be O(n), with n being the no. of rows in the table, because it needs to search the entire table for this data, and this is not something suitable or desired.

Now, one thing we can propose is let's use Binary Search Trees instead. And these actually reduce the time taken for range queries to O(logn). But it increases the normal read and write operations to O(logn) as well, which is worse than hash indexes.