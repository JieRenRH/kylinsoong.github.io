---
layout: blog
title:  "Teiid Results Caching Usage Example"
date:   2015-05-19 21:20:00
categories: teiid
permalink: /teiid-rs-cache
author: Kylin Soong
duoshuoid: ksoong2015051901
excerpt: Teiid Results Caching Example
---

Let's start from a performance comparison diagram

![Teiid rs cache]({{ site.baseurl }}/assets/blog/teiid-perf-resultset.png)

the left histogram is the query time without cache(1066 milliseconds), the right histogram is the query time with cache(almost 1 millisecond), we can get the conclusion: **enable Results Caching is 1000 times fast than disable caching**.

More details about Results Caching refer to [https://docs.jboss.org/author/display/TEIID/Results+Caching](https://docs.jboss.org/author/display/TEIID/Results+Caching).

## How to run  

With the following steps to run the performance comparison examples:

* **Step.1 Add test data to MySQL**

In my test I have insert 100 MB size data in Mysql `PERFTEST` table(CREATE TABLE PERFTEST(id INTEGER, col_a CHAR(16), col_b CHAR(40), col_c CHAR(40))).

> NOTE: int type is 4 bytes, char(n) is n bytes, so each row's size = 4 + 16 + 40 + 40, in other words, each rows size is 100 bytes.

So for insert 100 MB into Mysql, we need inser 1<<20(MB) rows. Query from Mysql Comman Line, the result:

~~~
> SELECT sum(table_rows), sum(data_length) from information_schema.TABLES WHERE table_name = 'PERFTEST';
+-----------------+------------------+
| sum(table_rows) | sum(data_length) |
+-----------------+------------------+
|         1048716 |        142262272 |
+-----------------+------------------+
~~~ 

* **Step.2 Create View in VDB map to Mysql Table**

The View in my test like:

~~~
<model name="Test" type="VIRTUAL">
	<metadata type="DDL"><![CDATA[
	CREATE VIEW PERFTESTVIEW (
	id long,
	col_a varchar,
	col_b varchar,
	col_c varchar
	)
	AS
	SELECT A.id, A.col_a, A.col_b, A.col_c FROM Accounts.PERFTEST AS A;
	]]>
	</metadata>
</model>
~~~

* **Query against PERFTESTVIEW**

Implement select all query 10 times with enable and disable User Query Cache.

Enable User Query Cache Result:

~~~
+-----------------------------------------+
| /*+ cache */ SELECT * FROM PERFTESTVIEW |
+-----------------------------------------+
|                                    1077 |
|                                      11 |
|                                       1 |
|                                       1 |
|                                       1 |
|                                       2 |
|                                       1 |
|                                       0 |
|                                       1 |
|                                       0 |
+-----------------------------------------+
~~~

Disable User Query Cache Result:

~~~
+-----------------------------------------+
|         SELECT * FROM PERFTESTVIEW      |
+-----------------------------------------+
|                                    1252 |
|                                    1226 |
|                                    1005 |
|                                     995 |
|                                    1005 |
|                                     983 |
|                                    1042 |
|                                    1089 |
|                                    1025 |
|                                    1047 |
+-----------------------------------------+
~~~

## How it work

In this section we discuss why 1000 times performance take place. As below sequence diagram, RequestWorkItem process first will get result from RssultSetCache, if result exist, get result from cache and return, this is the reson why 1000 times performance take place.

![Result From Cache]({{ site.baseurl }}/assets/blog/teiid-cache.png)

> Note that: more detailed logic about RequestWorkItem get results from cache please look at processNew() method in [RequestWorkItem.java](https://github.com/teiid/teiid/blob/master/engine/src/main/java/org/teiid/dqp/internal/process/RequestWorkItem.java) 

## Cached Virtual Procedure

Cached virtual procedure results are used automatically when a matching set of parameter values is detected for the same procedure execution. Use the Cache Hints can enable cache virtual procedure results, below as an example:

~~~
CREATE VIRTUAL PROCEDURE PERFTPROCE2()
AS
/*+ cache */
BEGIN 
	SELECT A.id, A.col_a, A.col_b, A.col_c FROM Accounts.PERFTEST AS A;
END
~~~

In my test there also is a `PERFTPROCE1()` which cache is diabaled, the test results show there also have thousands of times performance improve, the comparison result as below:

~~~
+--------------------+--------------------+
| call PERFTPROCE1() | call PERFTPROCE2() |
+--------------------+--------------------+
|        3622        |        3355        |
|        3236        |          1         |
|        3219        |          1         |
|        3233        |          1         |
|        3207        |          1         |
|        3207        |          1         |
|        3596        |          1         |
|        3192        |          1         |
|        3198        |          1         |
|        3180        |          1         |
+--------------------+--------------------+
~~~