
# 10 Tips for Optimizing MySQL Queries (That don't suck)

MYSQL 10 tips for optimizing MySQL queries. I couldn't read this and let it stand because this list is really, really bad. Some guy named Mike noted this, too. So in this entry I'll do two things: first, I'll explain why his list is bad; second, I'll present my own list which, hopefully, is much better. Onward, intrepid readers!
Why That List Sucks
1.	He's swinging for the top of the trees
The rule in any situation where you want to opimize some code is that you first profile it and then find the bottlenecks. Mr. Silverton, however, aims right for the tippy top of the trees. I'd say 60% of database optimization is properly understanding SQL and the basics of databases. You need to understand joins vs. subselects, column indices, how to normalize data, etc. The next 35% is understanding the performance characteristics of your database of choice. COUNT(*) in MySQL, for example, can either be almost-free or painfully slow depending on which storage engine you're using. Other things to consider: under what conditions does your database invalidate caches, when does it sort on disk rather than in memory, when does it need to create temporary tables, etc. The final 5%, where few ever need venture, is where Mr. Silverton spends most of his time. Never once in my life have I used SQL_SMALL_RESULT.
2.	Good problems, bad solutions
There are cases when Mr. Silverton does note a good problem. MySQL will indeed use a dynamic row format if it contains variable length fields like TEXT or BLOB, which, in this case, means sorting needs to be done on disk. The solution is not to eschew these datatypes, but rather to split off such fields into an associated table. The following schema represents this idea:
CREATE TABLE posts (
	id int unsigned not null auto_increment,
	author_id int unsigned not null,
	created timestamp not null,
	PRIMARY KEY(id)
);

CREATE TABLE posts_data (
	post_id int unsigned not null.
	body text,
	PRIMARY KEY(post_id)
);
3.	That's just...yeah
Some of his suggestions are just mind-boggling, e.g., "remove unnecessary paratheses." It really doesn't matter whether you do SELECT * FROM posts WHERE (author_id = 5 AND published = 1) or SELECT * FROM posts WHERE author_id = 5 AND published = 1. None. Any decent DBMS is going to optimize these away. This level of detail is akin to wondering when writing a C program whether the post-increment or pre-increment operator is faster. Really, if that's where you're spending your energy, it's a surprise you've written any code at all
My list
Let's see if I fare any better. I'm going to start from the most general.
1.	Benchmark, benchmark, benchmark!
You're going to need numbers if you want to make a good decision. What queries are the worst? Where are the bottlenecks? Under what circumstances am I generating bad queries? Benchmarking is will let you simulate high-stress situations and, with the aid of profiling tools, expose the cracks in your database configuration. Tools of the trade include supersmack, ab, and SysBench. These tools either hit your database directly (e.g., supersmack) or simulate web traffic (e.g., ab).
2.	Profile, profile, profile!
So, you're able to generate high-stress situations, but now you need to find the cracks. This is what profiling is for. Profiling enables you to find the bottlenecks in your configuration, whether they be in memory, CPU, network, disk I/O, or, what is more likely, some combination of all of them.
The very first thing you should do is turn on the MySQL slow query log and install mtop. This will give you access to information about the absolute worst offenders. Have a ten-second query ruining your web application? These guys will show you the query right off.
After you've identified the slow queries you should learn about the MySQL internal tools, like EXPLAIN, SHOW STATUS, and SHOW PROCESSLIST. These will tell you what resources are being spent where, and what side effects your queries are having, e.g., whether your heinous triple-join subselect query is sorting in memory or on disk. Of course, you should also be using your usual array of command-line profiling tools like top, procinfo, vmstat, etc. to get more general system performance information.
3.	Tighten Up Your Schema
Before you even start writing queries you have to design a schema. Remember that the memory requirements for a table are going to be around #entries * size of a row. Unless you expect every person on the planet to register 2.8 trillion times on your website you do not in fact need to make your user_id column a BIGINT. Likewise, if a text field will always be a fixed length (e.g., a US zipcode, which always has a canonical representation of the form "XXXXX-XXXX") then a VARCHAR declaration just adds a superfluous byte for every row.
Some people poo-poo database normalization, saying it produces unecessarily complex schema. However, proper normalization results in a minimization of redundant data. Fundamentally that means a smaller overall footprint at the cost of performance — the usual performance/memory tradeoff found everywhere in computer science. The best approach, IMO, is to normalize first and denormalize where performance demands it. Your schema will be more logical and you won't be optimizing prematurely.
4.	Partition Your Tables
Often you have a table in which only a few columns are accessed frequently. On a blog, for example, one might display entry titles in many places (e.g., a list of recent posts) but only ever display teasers or the full post bodies once on a given page. Horizontal vertical partitioning helps:
CREATE TABLE posts (
	id int unsigned not null auto_increment,
	author_id int unsigned not null,
	title varchar(128),
	created timestamp not null,
	PRIMARY KEY(id)
);

CREATE TABLE posts_data (
	post_id int unsigned not null,
	teaser text,
	body text,
	PRIMARY KEY(post_id)
);
The above represents a situation where one is optimizing for reading. Frequently accessed data is kept in one table while infrequently accessed data is kept in another. Since the data is now partitioned the infrequently access data takes up less memory. You can also optimize for writing: frequently changed data can be kept in one table, while infrequently changed data can be kept in another. This allows more efficient caching since MySQL no longer needs to expire the cache for data which probably hasn't changed.
5.	Don't Overuse Artificial Primary Keys
Artificial primary keys are nice because they can make the schema less volatile. If we stored geography information in the US based on zip code, say, and the zip code system suddenly changed we'd be in a bit of trouble. On the other hand, many times there are perfectly fine natural keys. One example would be a join table for many-to-many relationships. What not to do:
CREATE TABLE posts_tags (
	relation_id int unsigned not null auto_increment,
	post_id int unsigned not null,
	tag_id int unsigned not null,
	PRIMARY KEY(relation_id),
	UNIQUE INDEX(post_id, tag_id)
);
Not only is the artificial key entirely redundant given the column constraints, but the number of post-tag relations are now limited by the system-size of an integer. Instead one should do:
CREATE TABLE posts_tags (
	post_id int unsigned not null,
	tag_id int unsigned not null,
	PRIMARY KEY(post_id, tag_id)
);
6.	Learn Your Indices
Often your choice of indices will make or break your database. For those who haven't progressed this far in their database studies, an index is a sort of hash. If we issue the query SELECT * FROM users WHERE last_name = 'Goldstein' and last_name has no index then your DBMS must scan every row of the table and compare it to the string 'Goldstein.' An index is usually a B-tree (though there are other options) which speeds up this comparison considerably.
You should probably create indices for any field on which you are selecting, grouping, ordering, or joining. Obviously each index requires space proportional to the number of rows in your table, so too many indices winds up taking more memory. You also incur a performance hit on write operations, since every write now requires that the corresponding index be updated. There is a balance point which you can uncover by profiling your code. This varies from system to system and implementation to implementation.
7.	SQL is Not C
C is the canonical procedural programming language and the greatest pitfall for a programmer looking to show off his database-fu is that he fails to realize that SQL is not procedural (nor is it functional or object-oriented, for that matter). Rather than thinking in terms of data and operations on data one must think of sets of data and relationships among those sets. This usually crops up with the improper use of a subquery:
SELECT a.id, 
	(SELECT MAX(created) 
	FROM posts 
	WHERE author_id = a.id) 
AS latest_post
FROM authors a
Since this subquery is correlated, i.e., references a table in the outer query, one should convert the subquery to a join.
SELECT a.id, MAX(p.created) AS latest_post
FROM authors a
INNER JOIN posts p
	ON (a.id = p.author_id)
GROUP BY a.id
8.	Understand your engines
MySQL has two primary storange engines: MyISAM and InnoDB. Each has its own performance characteristics and considerations. In the broadest sense MyISAM is good for read-heavy data and InnoDB is good for write-heavy data, though there are cases where the opposite is true. The biggest gotcha is how the two differ with respect to the COUNT function.
MyISAM keeps an internal cache of table meta-data like the number of rows. This means that, generally, COUNT(*) incurs no additional cost for a well-structured query. InnoDB, however, has no such cache. For a concrete example, let's say we're trying to paginate a query. If you have a query SELECT * FROM users LIMIT 5,10, let's say, running SELECT COUNT(*) FROM users LIMIT 5,10 is essentially free with MyISAM but takes the same amount of time as the first query with InnoDB. MySQL has a SQL_CALC_FOUND_ROWS option which tells InnoDB to calculate the number of rows as it runs the query, which can then be retreived by executing SELECT FOUND_ROWS(). This is very MySQL-specific, but can be necessary in certain situations, particularly if you use InnoDB for its other features (e.g., row-level locking, stored procedures, etc.).
9.	MySQL specific shortcuts
MySQL provides many extentions to SQL which help performance in many common use scenarios. Among these are INSERT ... SELECT, INSERT ... ON DUPLICATE KEY UPDATE, and REPLACE.
I rarely hesitate to use the above since they are so convenient and provide real performance benefits in many situations. MySQL has other keywords which are more dangerous, however, and should be used sparingly. These include INSERT DELAYED, which tells MySQL that it is not important to insert the data immediately (say, e.g., in a logging situation). The problem with this is that under high load situations the insert might be delayed indefinitely, causing the insert queue to baloon. You can also give MySQL index hints about which indices to use. MySQL gets it right most of the time and when it doesn't it is usually because of a bad scheme or poorly written query.
10.	And one for the road...
Last, but not least, read Peter Zaitsev's MySQL Performance Blog if you're into the nitty-gritty of MySQL performance. He covers many of the finer aspects of database administration and performance.

