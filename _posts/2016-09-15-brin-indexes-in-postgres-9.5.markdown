---
title: "BRIN Indexes in Postgres 9.5"
author: cldellow
tags:
  - postgres
  - sql
---

_This was originally published on the [Sortable dev blog](https://dev.sortable.com/)._

We recently had a chance to explore one of Postgres 9.5's new features: block range indexes.

You're probably familiar with the most common (and default) form of index, the B-tree
index. B-tree indexes have entries for each row in your table, duplicating the data
in the indexed columns. This permits super-fast index-only scans, but can be
wasteful of disk space. The B-tree index also stores the data in sorted order, which permits
very fast lookups of single rows. It also stores the full location of each row in the
table, which can require a lot of work when running queries that return many rows.

A BRIN index, on the other hand, contains only the minimum and maximum values contained in
a group of database pages. It can rule out the presence of certain records, but that's
about it. As we'll see, that can be pretty powerful in the right circumstances.

## A tortured analogy

You can think of B-tree vs BRIN like the difference between an index at the back
of a book and labels on a filing cabinet [^1]. The book index is alphabetized
for ease of searching and is exhaustive: if it's not in the index, it's not in the book.
The filing cabinet labels, on the other hand, quickly narrow down where the record might be,
but you still have to check the drawer to find it.

## Our scenario

We have an analytics table that contains many, many rows broken up by day and customer.
We noticed that queries against the table were slower than expected. To find out why we ran the query 
through `EXPLAIN (ANALYZE ON, BUFFERS ON)`:

```
                                                                                                                                          QUERY PLAN                                                                                                                                           
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=455123.32..455126.64 rows=266 width=77) (actual time=1856.527..1897.616 rows=64567 loops=1)
   Group Key: t.day, t.country_code, t.device_type, t.site_id, t.ad_unit_name
   Buffers: shared hit=200138
   ->  Bitmap Heap Scan on revenue t ❶ (cost=4909.20..455030.36 rows=2656 width=77) (actual time=88.322..1402.759 rows=250205 loops=1)
         Recheck Cond: (site_id = ANY ('{870}'::integer[]))
         Rows Removed by Index Recheck: 4220407
         Filter: ((day >= '2016-08-01'::date) AND (day <= '2016-08-31'::date) AND (CASE WHEN ((direct_imps_raw > 0) OR (direct_imps > 0)) THEN 'Direct'::text WHEN ((house_imps_raw > 0) OR (house_imps > 0)) THEN 'House'::text ELSE 'Network'::text END = ANY ('{Network,Direct}'::text[])))
         Heap Blocks: exact=93002 lossy=106449
         Buffers: shared hit=200138 ❸
         ->  Bitmap Index Scan on revenue_site_id_idx ❷ (cost=0.00..4908.53 rows=265613 width=0) (actual time=65.172..65.172 rows=250205 loops=1)
               Index Cond: (site_id = ANY ('{870}'::integer[]))
               Buffers: shared hit=687 ❹
 Planning time: 0.270 ms
 Execution time: 1906.072 ms
(14 rows)
```

1,900 ms is not _terrible_, but it's not great.

We see (at ❶) that the query is only accessing one table and (at ❷) that it's using an index. It fetches 265,613 rows and aggregates those rows down to 64,567 groups. One thing that stands out (at ❸) is that the B-tree is touching a _lot_ of pages: 200,138 of them. Since each Postgres page is 8 KB, this is doing 1.6 GB of I/O! In this case, all of the pages were in Postgres's buffer cache, which really saves our hardware from the effects of that quantity of I/O. With cold caches, instead of 1,900 ms, this query takes close to 65 seconds to complete.

The summary suggests that either our rows are really big (200138 * 8192 / 265613 = approx. 6,200 bytes each), we have a lot of table bloat from dead rows, or the B-tree index has some huge overhead.

We ruled those potential causes out by using the following:

* `SELECT relname, relpages::bigint * 8192 / reltuples FROM pg_class WHERE relname = 'revenue'` to verify that our rows are about 200 bytes wide
* `VACUUM FULL` to ensure there are no dead rows
* observing that the index is only 400 MB total and, upon closer inspection (see ❹), only 687 of the 200,000 pages are related to the index

Since none of those techniques resulted in anything conclusive, perhaps our rows just weren't very efficiently packed on the disk? Luckily, Postgres exposes a special column named `ctid` that contains each row's physical page number and row offset. We'd really like it if a single page on disk had only rows pertaining to a single customer. Is that the case?

```
WITH sites_per_page AS (
  SELECT (ctid::text::point)[0]::bigint AS page_number, COUNT(DISTINCT site_id)
  FROM revenue
  GROUP BY 1
) SELECT avg(count) FROM sites_per_page;
         avg         
---------------------
 33.1613850916152352
```

It looks like we're loading the data so that rows related to a single customer are
scattered seemingly at random across the table. On average, a page contains data about 33 different
customers. With these findings in hand, we tinkered with our loader so that the data is ordered by day, and then by customer. Then we used the [`CLUSTER`](https://www.postgresql.org/docs/current/static/sql-cluster.html)
command to re-order the existing data to see if things were improved:

```
WITH sites_per_page AS (
  SELECT (ctid::text::point)[0]::bigint AS page_number, COUNT(DISTINCT site_id)
  FROM revenue
  GROUP BY 1
) SELECT avg(count) FROM sites_per_page;
         avg         
---------------------
 1.0202913372556230
```

Much better. How slow is that query now? `EXPLAIN` says:

```
                                                                                                                                          QUERY PLAN                                                                                                                                           
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=454004.73..454008.05 rows=265 width=77) (actual time=920.738..970.349 rows=64567 loops=1)
   Group Key: t.day, t.country_code, t.device_type, t.site_id, t.ad_unit_name
   Buffers: shared hit=6888
   ->  Bitmap Heap Scan on revenue t  (cost=4882.52..453912.26 rows=2642 width=77) (actual time=30.718..321.651 rows=250205 loops=1)
         Recheck Cond: (site_id = ANY ('{870}'::integer[]))
         Filter: ((day >= '2016-08-01'::date) AND (day <= '2016-08-31'::date) AND (CASE WHEN ((direct_imps_raw > 0) OR (direct_imps > 0)) THEN 'Direct'::text WHEN ((house_imps_raw > 0) OR (house_imps > 0)) THEN 'House'::text ELSE 'Network'::text END = ANY ('{Network,Direct}'::text[])))
         Heap Blocks: exact=6201
         Buffers: shared hit=6888
         ->  Bitmap Index Scan on revenue_site_id_idx  (cost=0.00..4881.85 rows=264189 width=0) (actual time=29.528..29.528 rows=250205 loops=1)
               Index Cond: (site_id = ANY ('{870}'::integer[]))
               Buffers: shared hit=687
 Planning time: 0.249 ms
 Execution time: 980.434 ms
(13 rows)

```

Now we get 980 ms instead of 1900 ms. Also, now that we're only fetching 6,888 pages, the cold cache case is down to
approximately 3 seconds.

# Isn't this blog post supposed to be about BRIN indexes?

So far we've only applied standard optimization techniques. Now that our
data is neatly ordered, can we do even better by using a BRIN index?

```
CREATE INDEX ON revenue USING BRIN(site_id) WITH (pages_per_range = 16);
```

`EXPLAIN` now shows:

```
                                                                                                                                          QUERY PLAN                                                                                                                                           
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=451524.29..451527.61 rows=265 width=77) (actual time=637.999..678.563 rows=64567 loops=1)
   Group Key: t.day, t.country_code, t.device_type, t.site_id, t.ad_unit_name
   Buffers: shared hit=7317
   ->  Bitmap Heap Scan on revenue t  (cost=2402.08..451431.82 rows=2642 width=77) (actual time=11.393..224.504 rows=250205 loops=1)
         Recheck Cond: (site_id = ANY ('{870}'::integer[]))
         Rows Removed by Index Recheck: 41707
         Filter: ((day >= '2016-08-01'::date) AND (day <= '2016-08-31'::date) AND (CASE WHEN ((direct_imps_raw > 0) OR (direct_imps > 0)) THEN 'Direct'::text WHEN ((house_imps_raw > 0) OR (house_imps > 0)) THEN 'House'::text ELSE 'Network'::text END = ANY ('{Network,Direct}'::text[])))
         Heap Blocks: lossy=7184
         Buffers: shared hit=7317
         ->  Bitmap Index Scan on revenue_site_id_idx  (cost=0.00..2401.42 rows=264189 width=0) (actual time=11.316..11.316 rows=71840 loops=1)
               Index Cond: (site_id = ANY ('{870}'::integer[]))
               Buffers: shared hit=133
 Planning time: 0.192 ms
 Execution time: 686.989 ms
(14 rows)
```

700ms. Nice. What changed? For one, the index is much smaller: where the B-tree index was 450 MB, the BRIN index is
only 840 KB. As a result, much less of the I/O is related to the index. So, despite our optimizations, we fetched more pages overall.
This page increase is due to the filing cabinet nature of BRIN indexes: our new BRIN index has snared some pages that don't contain
relevant rows. You can tune this behaviour via the `pages_per_range` parameter. Even so, navigating the BRIN index appears to be computationally cheaper than navigating a B-tree index, and the overall query time is decreased. The query is still kind of slow, but for the purposes of this article, we'll pretend we stopped optimizing it there. :)

You might be wondering -- how would a BRIN index have performed on our original, unordered table? To give you an idea of how the BRIN index would would work on the unordered data you can imagine that it's like a filing cabinet where every drawer is labelled 'A-Z', and therefore infer that the performance would be pretty horrible. And you'd be right:

```
                                                                                                                                          QUERY PLAN                                                                                                                                           
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=452626.88..452630.21 rows=266 width=77) (actual time=5590.931..5633.159 rows=64567 loops=1)
   Group Key: t.day, t.country_code, t.device_type, t.site_id, t.ad_unit_name
   Buffers: shared hit=522317
   ->  Bitmap Heap Scan on revenue  t  (cost=2412.76..452533.92 rows=2656 width=77) (actual time=49.201..5097.223 rows=250205 loops=1)
         Recheck Cond: (site_id = ANY ('{870}'::integer[]))
         Rows Removed by Index Recheck: 21112768
         Filter: ((day >= '2016-08-01'::date) AND (day <= '2016-08-31'::date) AND (CASE WHEN ((direct_imps_raw > 0) OR (direct_imps > 0)) THEN 'Direct'::text WHEN ((house_imps_raw > 0) OR (house_imps > 0)) THEN 'House'::text ELSE 'Network'::text END = ANY ('{Network,Direct}'::text[])))
         Heap Blocks: lossy=522184
         Buffers: shared hit=522317
         ->  Bitmap Index Scan on revenue_site_id_idx  (cost=0.00..2412.10 rows=265613 width=0) (actual time=48.912..48.912 rows=5221920 loops=1)
               Index Cond: (site_id = ANY ('{870}'::integer[]))
               Buffers: shared hit=133
 Planning time: 0.189 ms
 Execution time: 5641.325 ms
(14 rows)

```

It reads the index fast -- only 133 buffers -- and concludes that it has to read over 500,000 pages. It then discards 99% of the rows that it fetched. This is, as you can see, pretty slow.

## tl;dr

| Table order | Index type | Table size (MB) | Index size (MB) | # of buffers | Cold time (ms) | Warm time (ms) |
|:-----------:|:----------:|:---------------:|:---------------:|:------------:|:--------------:|:--------------:|
| Unordered   | B-tree     | 4,080           | 458             | 200,138      | 65,000         | 1,900          |
| Unordered   | BRIN       | 4,080           | 0.84            | 522,317      | 175,000        | 5,641          |
| Ordered     | B-tree     | 4,080           | 458             | 6,888        | 3,000          | 980            |
| Ordered     | BRIN       | 4,080           | 0.84            | 7317         | 2,600          | 700            |

BRIN indexes can speed things up a lot, but only if your data has a strong natural ordering to begin with. In our case, using a BRIN index was 500 times more space efficient than a B-tree index, and ran a bit faster too. We managed to improve our best case time by a factor of 2.6 and our worst case time by a factor of 30 with some simple optimizations.

Postgres can also apply multiple BRIN indexes (which results in what would be equivalent to a filing cabinet with two labels per drawer) for a single query. We ultimately achieved a 25% decrease in disk usage without decreasing query performance -- nice!

[^1]: Actually, as [pg_xocolatl](https://twitter.com/pg_xocolatl/status/776670328118542336) points out, the index at the back of a book is more like a GIN index since it has a 1-to-many mapping from key to page number. We did say this was a tortured analogy!
