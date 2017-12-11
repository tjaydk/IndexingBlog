## Always!! Like in ALWAYS, use indexes when creating queries that should retrieve ranged datasets

##### By Rune V. Zimsen, Ebbe V. Nielsen & Dennis M. Rønnebæk

![](https://upload.wikimedia.org/wikipedia/commons/thumb/c/cb/Postgres_Query.jpg/1200px-Postgres_Query.jpg)

***If you don't, you WILL suffer in the long run and have a solution that will not perform.***

***Make sure that you either use primary keys or adds indexes on query parameters that are used to define a set range.***

***This will increase performance significantly, especially if you aim to create software that will end up being popular.***



***SO what is the problem?***

When working with a relational database it is a very common task to retrieve datasets that are related in some range. So in order to retrieve this data you will need some parameter to sort the data by. Just like searching for a specific tuple in a relational database, using a primary key as the search parameter is the most sufficient way of retrieving the data. This is due to the primary key being indexed by default and stored in a tree structure which makes it very fast to search by. In the same way we would prefer to use some indexed parameter when we are defining our dataset range. When trying to retrieve a ranged dataset using a parameter that is not indexed will result in the database having to sort the data before being able to retrieve the dataset, thereby having to apply a sorting algorithm like heap sort, merge sort or quick sort. The latter having the issue where, if the data is already sorted or almost sorted, having a very bad performance, and therefor having to implement a shuffle mechanism in order to ensure decent performance..

***WHY is this a problem?***

If you have a lot of tuples in your database and then have to apply sorting before being able to retrieve the data it will have a noticeable impact on the performance of said query. Let's try to look at an example. Here we have a database that contains +1.000.000 posts and you want to retrieve the latest 30 post persisted in the database. Each tuple has a creation date attribute that we will use to order the data and then retrieve the 30 latest posts. We create the following query.

```sql
SELECT *
FROM posts AS p
JOIN users u
ON p.user_ref = u.id
LEFT JOIN votes_users_posts v
ON v.post_ref = p.id AND v.user_ref = 22
WHERE p.id < 3204334
ORDER BY p.created_at DESC -- < this value is not indexed
LIMIT 30;
```

In the above example we have a query that selects all attributes of the posts with some different joins and finally an ordering based on a created_at attribute in the post tuple which is not indexed. So the database has to sort all the tuples in the database based on the created_at value, which in average would mean an operation of O(n log n), and then afterwards retrieve the data. So when we have a massive amount of tuples in the database this will be an unnecessarily heavy operation and will cause overall bad performance and user experience in the long run. But luckily there is a fix.

***What can I do to change this?***

So in order to avoid bad performance and bad user experience when dealing with data retrieval you should ensure indexes on the parameters. As stated earlier there is already an index on a primary key, but you can add indexes on every parameter you like.  Here is a cheat sheet showing you how to create an index on an attribute in your database. *Note that the syntax and some functionality may differ depending on which DBMS you use. - The cheat sheet below is based on a MariaDB implementation of a relational database.*

```sql
-- reference: https://mariadb.com/kb/en/library/create-index/
CREATE [OR REPLACE] [ONLINE|OFFLINE] [UNIQUE|FULLTEXT|SPATIAL] INDEX 
  [IF NOT EXISTS] index_name
    [index_type]
    ON tbl_name (index_col_name,...)
    [WAIT n | NOWAIT]
    [index_option]
    [algorithm_option | lock_option] ...

index_col_name:
    col_name [(length)] [ASC | DESC]

index_type:
    USING {BTREE | HASH | RTREE}

index_option:
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'

algorithm_option:
    ALGORITHM [=] {DEFAULT|INPLACE|COPY}

lock_option:
    LOCK [=] {DEFAULT|NONE|SHARED|EXCLUSIVE}
```

When you create an index it is stored on the disk in order for the optimizer to quickly being able to retrieve the structure and use it to retrieve the data. In our query example from earlier we have a scenario where the created_at is a timestamp stating when the post is persisted, which means it follows the same sequence as the id which is a primary key. As we know, the primary key in our table is already indexed. So we don't even have to create a new index in this example in order to optimize the above query, we just have to change the **ORDER BY** operation to use the id instead.
```sql
SELECT *
FROM posts AS p
JOIN users u
ON p.user_ref = u.id
LEFT JOIN votes_users_posts v
ON v.post_ref = p.id AND v.user_ref = 22
WHERE p.id < 3204334
ORDER BY p.id DESC -- < this value is a PRIMARY KEY and therefore indexed by default
LIMIT 30;
```

So here we see the updated query, where we just changed the attribute in the **ORDER BY** operation from using the created_at attribute to the id, which was the primary key of the table and thereby eliminating the need to sort the data before hand.

***Why does this make my query faster?***

An index is actually just a data structure that is added to a column in a table. The most common data structure used is a binary tree, so that is what most DBMS creates when making an index. Another option is to use a hash table, but a disadvantage of that is that the index can't be sorted which makes **less-than** and **greater-than** operations impossible. So be sure to analyze what types of querying is needed on the table when introducing indexes. 

In the above query example a hash table index structure would not be optimal as we want to retrieve a ranged dataset based on the latest id. Hash tables on the other hand would be great when retrieving a single tuple with a already know value of the indexed attribute.  

![](https://devjeetr.files.wordpress.com/2012/04/dumptree-dot.png)

*Example of a binary tree structure*

What makes a binary tree structure fast is the way it works. The tree is build of sub trees made up of nodes. A parent node and a left and a right child node. The left child node will hold a value less than the parent node, and the node on the right will hold a value greater than the parent node. This makes up a subtree and the whole tree is build of these subtrees. So if we go to the root of the tree, we know that each subtree in the left branch will hold values less than the root, and YES, you guessed it, all the values in the right branch will hold values greater. A specific tree type called a RED-BLACK Binary Tree will ensure a balanced search tree and ensure best performance. But don't worry. The tree implementation is handle by the DBMS.

The other thing that is added when making an index is a pointer to the corresponding tuple in the table. As the index doesn't store any other information about the tuple, we need to have this pointer to get the rest of the data. So we have the key, which is the value we put an index on and the value which is a pointer to a place in memory where the rest of the tuple is.

So now when a query asks for an indexed value it will go into the data structure that holds the index, retrieve the pointer and then find the place in memory to retrieve the rest of the data of that specific tuple. 

***So why not put indexes on every attribute?***

Don't just add indexes to every attribute just because it will make retrieving data faster, because there a some disadvantages to adding indexes to attributes. It obviously takes up space as you need to create the entire data structure that will hold the index. And as the table grows bigger, more extra space is needed to hold the index. It also make's add, update and delete operations in the database more costly, as now when a tuple is added, updated or deleted, the same operation has to be done on the index data structure. That's why there is a general rule of not indexing something that is not queried frequently.

#### *That's it !!*

From introducing the index in our **ORDER BY** operation we are able to handle HUGE amounts of data and retrieve data extremely fast as we are using the tree structure and therefore we are lowering the cost of the operation to approximately O(log n) (reference: https://en.wikipedia.org/wiki/Binary_search_tree). We are now ready to approach the world with renewed confidence.



