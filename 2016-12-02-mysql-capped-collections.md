---
layout: post
title:  "Capped Collections in MySQL"
date:   2016-12-02 16:54:57 +0100
---
I currently work for a German social networking site for business professionals
called [XING](https://www.xing.com).  As you can probably imagine, a member
base of double-digit millions of users produces some pretty large amounts of data.
Fortunately, for the Engineering Team this provides us with a never-ending source
of interesting challenges to solve.

Not too long ago we were given the task to
evaluate various technical solutions for a feature which would potentially cause
up to 50 million records to be written per day.  What I'd like to show you is a
proof of concept that we came up with for MySQL.

For the sake of simplicity, I'll model the problem as follows:

  * There are several million hens.
  * Each hen has a basket that it lays eggs into.
  * Some hens lay eggs faster than others.
  * A basket can only hold a specific amount of eggs.

Since we're dealing with such large numbers of hens and baskets we decided to not
go for the periodic emptying approach typical of a cronjob.  We wanted to build
a self sustaining system so that the application doesn't have to worry about cleaning
up.  The first thing that came to mind was something similar to Mongodb's
[Capped Collections](https://docs.mongodb.com/manual/core/capped-collections/).
Unfortunately, MySQL doesn't provide such a feature, but with a bit of creativity
we can build something that behaves very similarly.

Let's start with our baskets table:

```sql
CREATE TABLE `baskets` (
  `basket_id` int(10) unsigned NOT NULL,
  `egg_id` int(10) unsigned NOT NULL,
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
  PRIMARY KEY  (`basket_id`, `egg_id`),
  KEY `basket_id` (`basket_id`,`created_at`)
);
```

Now our hens can lay eggs in their baskets to their heart's content:

```sql
mysql> INSERT INTO baskets (basket_id, egg_id) VALUES (42, 1);
```

In order to know when the baskets are full we'll need to know how many eggs are in them.
We can keep track of the number of eggs in each basket by counting what's being laid
in a summary table:

```sql
CREATE TABLE `basket_count` (
  `basket_id` int(10) unsigned NOT NULL,
  `count` int(10) unsigned NOT NULL,
  PRIMARY KEY  (`basket_id`)
);
```

The state of the counters will be kept in sync by adding some triggers to the baskets table:

```sql
mysql> CREATE TRIGGER baskets_ai AFTER INSERT ON baskets
         FOR EACH ROW
           INSERT INTO basket_count VALUES (new.basket_id, 1) ON DUPLICATE KEY UPDATE count=count+1;
mysql> CREATE TRIGGER baskets_ad AFTER DELETE ON baskets
         FOR EACH ROW
           UPDATE basket_count SET count=count-1 WHERE basket_id=old.basket_id;
```

To keep the baskets from filling up we can execute a delete statement that will
remove the oldest entries from the table by making use of `LIMIT`:

```sql
mysql> DELETE FROM baskets WHERE basket_id=42 ORDER BY created_at LIMIT 1;
```

At first glance you might think we're almost done.  We'd like to limit the amount
of eggs in the basket to an even dozen so we can just add a trigger to the
`basket_count` table that executes a delete when the counter exceeds our specified mark:

```sql
mysql> DELIMITER $$
mysql> CREATE TRIGGER basket_count_au AFTER UPDATE ON basket_count
         FOR EACH ROW
           BEGIN
             IF new.count > 12 THEN BEGIN
               delete from baskets where basket_id=new.basket_id order by created_at limit 1;
             END; END IF;
           END$$
mysql> DELIMITER ;
```

If you try to insert some more rows everything goes fine until we reach our limit:

```sql
mysql> INSERT INTO baskets (basket_id, egg_id) VALUES (42, 1), (42, 2), (42, 3), (42, 4), (42, 5), (42, 6), (42, 7), (42, 8), (42, 9), (42, 10), (42, 11), (42, 12);
mysql> INSERT INTO baskets (basket_id, egg_id) VALUES (42, 13);
```

You'll quickly learn that MySQL complains:

```
ERROR 1442 (HY000): Can't update table 'baskets' in stored function/trigger because it is already used by statement which invoked this stored function/trigger.
```

The reason for this is that we've placed a trigger on `basket_count` which is a
summary table for `baskets`.  When `basket_count` is updated it attempts to update
`baskets` again and we've created an infinite loop.  To work around this we need
to resolve the problem of the recursive updates.  By cleverly adding another table
to act as a proxy we can now bind our clean-up trigger to its updates, thereby
eliminating the interdependency between `baskets` and `basket_count`.

```sql
CREATE TABLE `baskets_proxy` (
  `basket_id` int(10) unsigned NOT NULL,
  `egg_id` int(10) unsigned NOT NULL,
  PRIMARY KEY  (`basket_id`, `egg_id`)
) ENGINE=BLACKHOLE;
```

MySQL's BLACKHOLE storage engine is essentially a "no-op."  Nothing is persisted
and all queries return the empty set.  However, BLACKHOLE tables _can_ have triggers.
With this information in mind, let's start over.  First we'll need to truncate our
tables and drop the existing triggers we created:

```sql
mysql> TRUNCATE baskets;
mysql> TRUNCATE basket_count;
mysql> DROP TRIGGER baskets_ai;
mysql> DROP TRIGGER baskets_ad;
mysql> DROP TRIGGER basket_count_au;
```

Next we'll add triggers to `baskets_proxy` which will keep the summary table up-to-date.
We need to make sure we proxy the original inserts and deletes to `baskets` as well:

```sql
mysql> DELIMITER $$
mysql> CREATE TRIGGER baskets_proxy_ai AFTER INSERT ON baskets_proxy
         FOR EACH ROW
           BEGIN
             INSERT INTO baskets (basket_id, egg_id) VALUES (new.basket_id, new.egg_id);
             INSERT INTO basket_count VALUES (new.basket_id, 1) ON DUPLICATE KEY UPDATE count=count+1;
           END$$
mysql> CREATE TRIGGER baskets_proxy_ad AFTER DELETE ON baskets_proxy
         FOR EACH ROW
           BEGIN
             DELETE FROM baskets WHERE basket_id=old.basket_id AND egg_id=old.egg_id;
             UPDATE basket_count SET count=count-1 WHERE basket_id=old.basket_id;
           END$$
mysql> DELIMITER ;
```

Now we can insert rows as before but using the proxy table instead:

```sql
mysql> INSERT INTO baskets_proxy (basket_id, egg_id) VALUES (42, 1);
```

Now let's set up a trigger to delete the oldest eggs once we reach 12.  We also
need to make sure that our counter doesn't get out of sync when we delete rows
from `baskets`.  The second IF-statement ensures that the counter in our summary
table doesn't exceed the maximum.

```sql
mysql> DELIMITER $$
mysql> CREATE TRIGGER basket_count_bu BEFORE UPDATE ON basket_count
         FOR EACH ROW
           BEGIN
             IF new.count > 12 THEN
               delete from baskets where basket_id=new.basket_id order by created_at limit 1;
             END IF;
             IF new.count >= 12 THEN
               SET new.count = 12;
             END IF;
           END$$
mysql> DELIMITER ;
```

Now when we insert more than a dozen rows we can see that the `baskets` table
only contains the most recent dozen eggs inserted:

```sql
mysql> INSERT INTO baskets_proxy (basket_id, egg_id) VALUES (42, 2), (42, 3), (42, 4), (42, 5), (42, 6), (42, 7), (42, 8), (42, 9), (42, 10), (42, 11), (42, 12), (42, 13);
mysql> SELECT * FROM baskets;
+-----------+--------+---------------------+
| basket_id | egg_id | created_at          |
+-----------+--------+---------------------+
|        42 |      2 | 2016-12-02 14:22:06 |
|        42 |      3 | 2016-12-02 14:22:06 |
|        42 |      4 | 2016-12-02 14:22:06 |
|        42 |      5 | 2016-12-02 14:22:06 |
|        42 |      6 | 2016-12-02 14:22:06 |
|        42 |      7 | 2016-12-02 14:22:06 |
|        42 |      8 | 2016-12-02 14:22:06 |
|        42 |      9 | 2016-12-02 14:22:06 |
|        42 |     10 | 2016-12-02 14:22:06 |
|        42 |     11 | 2016-12-02 14:22:06 |
|        42 |     12 | 2016-12-02 14:22:06 |
|        42 |     13 | 2016-12-02 14:22:06 |
+-----------+--------+---------------------+
12 rows in set (0.00 sec)

mysql> SELECT * FROM basket_count;
+-----------+-------+
| basket_id | count |
+-----------+-------+
|        42 |    12 |
+-----------+-------+
1 row in set (0.00 sec)
```


Voila!  Capped collections in MySQL!
