---
layout: post
title:  "Chicken or the Egg?"
date:   2016-11-28 16:54:57 +0100
published: false
---
I currently work for a German social networking site for business professionals
called [XING](https://www.xing.com).  As you can probably imagine, a member
base of double-digit millions of users produce some pretty large amounts of data.
Fortunately, for the Engineering Team this provides us with a never-ending source
of interesting challenges to solve.  Not too long ago we were given the task to
evaluate various technical solutions for a feature which would potentially cause
up to 50 million records to be written to our MySQL servers per day.

For the sake of simplicity and discretion, I'll model the problem as follows:

  * We have several million hens.
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
  PRIMARY KEY  (`basket_id`, `egg_id`)
);
```

Now our hens can lay eggs in their baskets until their hearts are content:

```sql
mysql> INSERT INTO baskets (basket_id, egg_id) VALUES (42, 12345);
```

In order to know when the baskets are full we'll need to know how many eggs are in them.
We can keep track of the number of eggs in each basket by counting what's being laid
in a separate table:

```sql
CREATE TABLE `basket_count` (
  `basket_id` int(10) unsigned NOT NULL,
  `cnt` int(10) unsigned NOT NULL,
  PRIMARY KEY  (`basket_id`)
);
```

The state of the counters will be kept in sync by adding some triggers to the baskets table:

```sql
mysql> CREATE TRIGGER baskets_ai AFTER INSERT ON baskets
         FOR EACH ROW
           INSERT INTO basket_count VALUES (new.basket_id, 1) ON DUPLICATE KEY UPDATE cnt=cnt+1;
mysql> CREATE TRIGGER baskets_ad AFTER DELETE ON baskets
         FOR EACH ROW
           UPDATE basket_count SET cnt=cnt-1 WHERE basket_id=old.basket_id;
```

Now at first glance you might think we're almost done.  We'll just add a trigger to the
basket_count table that executes a delete when the counter exceeds our specified mark:

```sql
mysql> CREATE TRIGGER basket_count_au AFTER UPDATE ON basket_count
         BEGIN
           IF new.cnt > 10 THEN
             DELETE FROM baskets WHERE basket_id=new.basket_id ORDER BY egg_id LIMIT 5
           END IF;
         END
```

It's important to note that we've made the assumption that our eggs' ids come from a sequence of ever-increasing numbers.  The smaller the id, the older the egg.

You'll quickly learn that MySQL doesn't like that:

```sql
mysql> INSERT INTO baskets (basket_id, egg_id) VALUES (42, 123456);
```

To work around this we need to resolve the problem of the recursive updates.  By putting another table in front of the others to act as a delegator we can now bind our clean-up trigger to a third independent party:

```sql
CREATE TABLE `basket_manager` (
  `basket_id` int(10) unsigned NOT NULL,
  `egg_id` int(10) unsigned NOT NULL,
  PRIMARY KEY  (`basket_id`, `egg_id`)
) ENGINE=BLACKHOLE;
```

```sql
mysql> CREATE TRIGGER basket_manager_ai AFTER INSERT ON basket_manager
         FOR EACH ROW
           BEGIN
             IF new.cnt > 10 THEN
               DELETE FROM baskets WHERE basket_id=new.basket_id LIMIT 5
             END IF
           END;
```
