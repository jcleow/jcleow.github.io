
## Optimizing slow queries in a rapidly growing Postgres Table - Part 1

## A. Problem:

- At `$work`, there was a tooling created to help internal sales team personnels to manage new clients.
- The solution attempted to upsert ~50k of potential new client contacts for each of our internal sales users which resulted in ~1.3m rows in a table called `crm_contact` table
- This led to extremely slow `count` queries, (the count was a requirement in the product specifications)
    - Anecdotally the queries could have gone up from 1-3seconds, which is problematic.


## B. Table Setup
* The data exists in a Postgres DB.
* In this post, I simplify the schema and relations of the table here for ease of understanding, and also for security purposes. But the key ideas can still be gleaned from the rest of the articles below.
* Naturally, the table and relevant variable names/details were also renamed for the above purposes.

## C. Diagnosis

```psql
explain analyze (
    SELECT
        COUNT(*)
    FROM
        crm_contact contact
    LEFT JOIN
        crm_agent agent_info
    ON
        agent_info_id = agent_info.id
    WHERE user_id = '{user_id}' AND contact.is_active = True
    AND agent_status = 'active'
);
```


```psql
QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=47705.28..47705.29 rows=1 width=8) (actual time=741.154..741.159 rows=1 loops=1)
   ->  Hash Join  (cost=6022.15..47615.08 rows=36077 width=0) (actual time=392.947..737.100 rows=36267 loops=1)
         Hash Cond: (contact.agent_info_id = agent_info.id)
         ->  Bitmap Heap Scan on crm_contact contact  (cost=3085.96..44542.25 rows=52052 width=23) (actual time=338.517..643.809 rows=52221 loops=1)
               Recheck Cond: (user_id = '{user_id}'::text)
               Filter: is_active
               Heap Blocks: exact=22988
               ->  Bitmap Index Scan on ix_crm_contact_user_id_is_active  (cost=0.00..3072.95 rows=52052 width=0) (actual time=322.048..322.049 rows=56139 loops=1)
                     Index Cond: ((user_id = '{user_id}'::text) AND (is_active = true))
         ->  Hash  (cost=2483.76..2483.76 rows=36194 width=23) (actual time=54.325..54.327 rows=36267 loops=1)
               Buckets: 65536  Batches: 1  Memory Usage: 2460kB
               ->  Seq Scan on crm_agent_info agent_info  (cost=0.00..2483.76 rows=36194 width=23) (actual time=0.013..33.944 rows=36267 loops=1)
                     Filter: (agent_status = 'active'::text)
                     Rows Removed by Filter: 15954
 Planning Time: 0.819 ms
 Execution Time: 741.273 ms
(16 rows)
```

As I can see here in the query plan, postgres’ query planner uses a hash join to join between `crm_contact` and `crm_agent_info` which involves

- iterating through all the records of both the inner and outer table to create an in-memory (or partially on disk) hash table.
- For the inner table (`crm_contact`) - (~643ms) this scan is done in 2 parts:
    - Bitmap Index Scan
        - The db actually first scans `ix_crm_contact_user_id_is_active` to find out what data it needs to fetch (but does not fetch the data itself from the table!)
        - It constructs a bitmap of the potential row locations
        - feeds It then this data to the parent Bitmap Heap Scan
    - Bitmap Heap Scan over `crm_contact`
        - A bitmap heap scan is probably chosen because the total number of qualifying rows is at 52k, which the DB probably can’t fit everything into memory
        - [https://pganalyze.com/docs/explain/scan-nodes/bitmap-index-scan#:~:text=Instead of producing the rows,grabbing data page by page](https://pganalyze.com/docs/explain/scan-nodes/bitmap-index-scan#:~:text=Instead%20of%20producing%20the%20rows,grabbing%20data%20page%20by%20page).
- This can be quite an expensive operation if I are fetching only a few rows.
- However, I are attempting to fetch 52k rows out of 1.3m rows which is about 5.2% of total rows.
- For the outer table (`crm_agent_info`) (~33.944ms)
    - It is performing a sequential scan over 36k rows(where the total number of rows on `crm_agent_info` was about 52k)
    - This was at first expected because the query planner usually uses sequential scan to fetch a large number of rows in a table.

As I can see the count was particularly slow in the `crm_contact` bitmap heap scan portion. This is to be expected as databases like postgresql is not good at performing aggregates because it must scan through all the rows to fetch the count.

I.e fetching large amounts of data at once means it needs to traverse the tree


# D. Potential strategies considered used to optimize for Count speed

Here are some of the options that I considered to optimize for count speed:

1. Increasing the `work_mem`
2. Adjusting table `statistics`
3. Use an estimated count
4. Remove the count from the app
5. Limit the amount of counting to 1000 - 2000 rows
6. Are there any missing indices?
7. Partitioning the table

## D.1. Increasing the work_mem

### What is `work_mem`?

- Work mem refers to the amount of memory each query may use for sorting/storing a temporary hash table
- The default `work_mem` on postgres is 4MB, and it can be seen by doing `SHOW work_mem`

### **Why does `work_mem` matter?**

- Queries against a large dataset like `crm_contact` requires alot of memory, especially the hash join operation that was being performed
- If sorting or the hash table needs more memory than permited by `work_mem`, then PostgreSQL will use temp files on disk to perform such operations. Since Disk IO is alot slower than memory IO, such heavy queries may become increasingly slow.

### Is our query slow because it is using on disk storage?

- Usually a clear indicator in the `EXPLAIN ANALYZE` will indicate that disk storage is used when I see things like `Sort Method: external merge Disk: xxxx KB`
- However, I did not see such lines in the explain analyze so it was abit dubious

### Pitfalls of tuning work_mem

- Setting too high a `work_mem` may mean any query may consume too much memory, leading to an out of memory situation
- I can mitigate this by setting `hash_mem_multiplier` in PG 13, which allows us to increase the maximum amount of memory available to hash-based operations( just like our hash join operation above).
- Hence for sort operations, it will continue to use the default 4MB work_mem, and this helps us isolate at least part of the memory consumption
    - Source: [https://www.pgmustard.com/blog/work-mem#:~:text=This is done by adjusting,starting point for most people](https://www.pgmustard.com/blog/work-mem#:~:text=This%20is%20done%20by%20adjusting,starting%20point%20for%20most%20people).

### Attempt & Conclusion

- Regardless, I tried playing around with the `work_mem` for a little bit to see if it helped
- I tried to increase the work mem to **16MB**  and then more in the single session, but it did not help significantly with the query speed.
    - The `work_mem` can be set by doing something like : `SET work_mem = '16MB'`
- I suspect this is because the hash table actually does not use alot of memory, despite `crm_contact` having a large table size of ~1GB at point of writing.
- Furthermore, the additional memory used of ~2.4MB to hash `crm_agent_info` to the hash table wasn’t really resulting in an disk storage usage, at least according to explain analyze.

References: https://andreigridnev.com/blog/2016-04-16-increase-work_mem-parameter-in-postgresql-to-make-expensive-queries-faster/


## D.2. Adjusting Statistics used by Planner

### Table Statistics

- As evident in the query planner, it needs to **estimate** the number of rows retrieved by a query in order to make good choices of query plans
- A critical component of statistics is the total number of entries in each table and index, as well as number of disk blocks occupied by each table and index.
- Such information is stored in `pg_class`, where it is stored in `reltuples` and `relpages`
- However, these statistics are not updated on the fly, hence they usually contain some out of date values
- Can be updated by `VACUUM`, `ANALYZE` or `CREATE INDEX`

> `ALTER TABLE SET STATISTICS` command, or globally by setting the **[default_statistics_target](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-DEFAULT-STATISTICS-TARGET)** configuration variable. The default limit is presently 100 entries.

Raising the limit **might** allow more accurate planner estimates to be made, particularly for columns with irregular data distributions, at the price of consuming more space in `pg_statistic` and slightly more time to compute the estimates. Conversely, a lower limit might be sufficient for columns with simple data distributions.
—  https://www.postgresql.org/docs/current/planner-stats.html
>

### Attempt & Conclusion

- I tried to increase the statistics for `crm_contact`  as inspired by this stackoverflow **[answer](https://stackoverflow.com/a/8108807/4973124) by setting the statistics**  on `user_id` table to play around, but there were no notable speed ups
- Hypothesis on why this might not have worked might be because I are increasing the statistics on the wrong column, or the data is quite evenly distributed already.
- This would have required much indepth understanding of the data distribution to make a more informed statistics collection, but I did not do it at the point in time.
    - [https://aws.amazon.com/blogs/database/understanding-statistics-in-postgresql/#:~:text=For the case where there,are sampled from each table](https://aws.amazon.com/blogs/database/understanding-statistics-in-postgresql/#:~:text=For%20the%20case%20where%20there,are%20sampled%20from%20each%20table).

### Other things that I would like to try but did not:

- Another suggestion on stackoverflow was to increase the overall statistics collected for the DB:
    - https://dba.stackexchange.com/a/278414
- Extended statistics
    - https://build.affinity.co/how-I-used-postgres-extended-statistics-to-achieve-a-3000x-speedup-ea93d3dcdc61


References: [https://aws.amazon.com/blogs/database/understanding-statistics-in-postgresql/#:~:text=For the case where there,are sampled from each table](https://aws.amazon.com/blogs/database/understanding-statistics-in-postgresql/#:~:text=For%20the%20case%20where%20there,are%20sampled%20from%20each%20table).



### D.3. Using an estimated count

Source: https://www.citusdata.com/blog/2016/10/12/count-performance/#dup_counts_estimated

- If I are willing to use an estimated rather than exact count, I can get fast reads with no insert degradation
- I make use of the estimated stats from either the  [stats collector](https://www.postgresql.org/docs/9.5/monitoring-stats.html) or the [autovacuum](https://www.postgresql.org/docs/9.5/routine-vacuuming.html#AUTOVACUUM) daemon i.e `reltuples`

> *Remember that reltuples isn't the estimate that the planner actually uses; the planner uses reltuples/relpages multiplied by the current number of pages.*
>
Conclusion - did not use this as product requirement did not want an estimate.


### D.4. Limiting the Count

- Technically counting requires us to scan all the rows that meet the required criteria anyway, so limiting the count instantly improves performance

```sql
EXPLAIN ANALYZE(SELECT COUNT(*)
FROM (
    SELECT 1
    FROM crm_contact contact
    LEFT JOIN crm_agent_info agent_info ON agent_info_id = agent_info.id
    WHERE user_id = ''{user_id}'' AND contact.is_active = TRUE
    AND agent_status = 'active'
    LIMIT 1000
) AS subquery);

                                                                                           QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=2374.10..2374.11 rows=1 width=8) (actual time=39.088..39.090 rows=1 loops=1)
   ->  Limit  (cost=0.41..2361.60 rows=1000 width=4) (actual time=6.734..38.992 rows=1000 loops=1)
         ->  Nested Loop  (cost=0.41..85543.65 rows=36229 width=4) (actual time=6.733..38.899 rows=1000 loops=1)
               ->  Seq Scan on crm_contact contact  (cost=0.00..55437.49 rows=52272 width=23) (actual time=6.696..29.305 rows=1412 loops=1)
                     Filter: (is_active AND (user_id = ''{user_id}''::text))
                     Rows Removed by Filter: 95884
               ->  Index Only Scan using ix_crm_agent_info_id_agent_status on crm_agent_info agent_info  (cost=0.41..0.58 rows=1 width=23) (actual time=0.006..0.006 rows=1 loops=1412)
                     Index Cond: ((id = contact.agent_info_id) AND (agent_status = 'active'::text))
                     Heap Fetches: 921
 Planning Time: 0.677 ms
 Execution Time: 39.138 ms
(11 rows)
```

This is a huge speed up as compared to `~800ms` of execution time!

- Product wise, at least on the app, there is no explicit page numbers, it would be difficult for users to physically scroll to 1000 contacts.
- However, this would also mean I need to decide on which filter combinations do I impose such a limit, which results in a similar problem as estimating the count

### D.5 Are there missing indices?
- By experience, index scans are good for performant for around 5% number of rows being returned, but in actual fact if its about 1-3%
- In our case I are fetching close to 5%, so sequential and index scan doesn’t really cut it so thats why I are doing a hash join with bitmap heap scans + bitmap index scans

However, these solutions were only partially effective to the problem that I was trying to solve for. I wanted to speed up the DB query while retaining the count feature for the product.


Stay tuned for Part 2 where I decided to use Partitioning on the tables how it helped in achieving this goal.