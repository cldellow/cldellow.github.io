---
layout: post
title:  "Query Parquet files in SQLite"
date:   2018-06-21 20:36:10 -0500
author: cldellow
tags:
  - postgres
  - sqlite
  - parquet
---

_Hello, reader! If you work with big files and S3, you may find my [s3patch](https://s3patch.com) utility useful._

---
_This blog post explains the motivation for the creation of a SQLite virtual table extension for Parquet files. Go to [cldellow/sqlite-parquet-vtable](https://github.com/cldellow/sqlite-parquet-vtable) if you just want the code._

I was playing around with a project to visualize data from the 2016 Canada census. The raw data from Stats Canada is a 1291 MB CSV [^1]. The CSV has 12,000,000 rows, each capturing metrics for men and women in a census area. Over 5,000 census areas are covered, with 2,200 dimensions in each. I wanted to find out what percentage of people travelled by bicycle in three particular cities.

My end goal is to make a standalone web site that lets you explore the data. Ideally, it'd be compact and could run on a dirt-cheap shared web host.

For my purposes, a 1291 MB CSV was too cumbersome. In addition to being large, `grep`ping for a single record is slow:

```
cldellow@furfle:~$ time grep Dawson.Creek.*Bicycle census.csv
2016,"5955014","3","Dawson Creek",6.3,8.3,"00101","CY","5955014","Bicycle",1935,,25,25,0

real    0m0.858s
```

So what are my options? I considered my three favourite database technologies: Postgres, SQLite and parquet files.

## Postgres

Postgres is a great general purpose tool. I always start with it.

### Stringly typed

Let's start by shoving the data into Postgres as-is, treating everything as a string. This is quick
to implement:

```
CREATE TABLE strings(
  year text, geo_code_por text, geo_level text, geo_name text, gnr text,
  gnr_lf text, data_quality_flag text, csd_type_name text, alt_geo_code text,
  profile text, profile_id text, notes text, total text, male text,
  female text
);

COPY strings(year, geo_code_por, geo_level, geo_name, gnr, gnr_lf,
  data_quality_flag, csd_type_name, alt_geo_code, profile, profile_id,
  notes, total, male, female )
FROM '/home/cldellow/src/csv2parquet/census.tsv';
```

`\d+` shows that this creates a 1438 MB table. We can query the table, casting strings to numeric datatypes as needed:

```
postgres=# WITH inputs AS (
  SELECT
    geo_name,
    CASE WHEN profile_id = '1930' THEN 'total' ELSE 'cyclist' END AS mode,
    female,
    male
  FROM strings
  WHERE
    profile_id IN ('1930', '1935') AND
    csd_type_name = 'CY' AND
    geo_name IN ('Victoria', 'Dawson Creek', 'Kitchener')
)
SELECT
  total.geo_name,
  cyclist.male,
  cyclist.female,
  (100.0 * cyclist.male::int / total.male::int)::numeric(4,2) AS pct_male,
  (100.0 * cyclist.female::int / total.female::int)::numeric(4,2) AS pct_female
FROM inputs AS total
JOIN inputs AS cyclist USING (geo_name)
WHERE total.mode = 'total' AND cyclist.mode = 'cyclist';

   geo_name   | male | female | pct_male | pct_female 
--------------+------+--------+----------+------------
 Dawson Creek | 25   | 0      |     0.86 |       0.00
 Kitchener    | 905  | 280    |     1.51 |       0.51
 Victoria     | 2650 | 2130   |    12.57 |       9.73
(3 rows)

Time: 723.383 ms
```

Not bad! No surprise: my rural hometown of Dawson Creek has almost no cyclists, the university town I live in has some, and the hippie enclave of Victoria has double digit percentages.

The query is a bit slow, though -- almost a second. If we add an index with `CREATE INDEX ON strings(geo_name)`, the query takes only 10ms -- but the index adds an extra 400 MB!

### Strongly typed

The CSV is denormalized. As such, it repeats a lot of the same values. For example, the string `Dawson Creek` appears 2,247 times. `Total - Lone-parent census families in private households - 100% data` appears 5,469 times. As you can imagine, that's pretty wasteful. A better approach would be to normalize the data so that, rather than storing thousands of copies of the same value, we store one value and just point to it.

Usually, you do this by creating separate lookup tables for each set of values. That's a lot of work for a blog post, so I'm going to compromise by using Postgres's [`ENUM`](https://www.postgresql.org/docs/current/static/datatype-enum.html) types. I'll also start storing numeric values as numeric values, rather than strings.

I'm lazy, so we'll bootstrap the enums and strongly-typed table from the existing `strings` table:

```
DO $$
BEGIN
  EXECUTE (
        SELECT format('CREATE TYPE geo_name AS ENUM (%s)',
                      string_agg(DISTINCT quote_literal(geo_name), ', '))
        FROM strings);

  EXECUTE (
        SELECT format('CREATE TYPE profile AS ENUM (%s)',
                      string_agg(DISTINCT quote_literal(md5(profile)), ', '))
        FROM strings);
END 
$$;

CREATE TABLE typed AS SELECT
  year::smallint,
  geo_code_por::int,
  geo_level::smallint,
  geo_name::geo_name,
  (CASE WHEN gnr = '' THEN NULL ELSE gnr END)::real AS gnr,
  (CASE WHEN gnr_lf = '' THEN NULL ELSE gnr_lf END)::real AS gnr_lf,
  data_quality_flag,
  csd_type_name,
  alt_geo_code,
  md5(profile)::profile,
  profile_id::smallint,
  (CASE WHEN notes = '' THEN NULL ELSE notes END)::smallint AS notes,
  (CASE WHEN total IN ('..', '...', 'x', 'F') THEN NULL ELSE total END)::real AS total,
  (CASE WHEN male IN ('..', '...', 'x', 'F') THEN NULL ELSE male END)::real AS male,
  (CASE WHEN female IN ('..', '...', 'x', 'F') THEN NULL ELSE female END)::real AS female
FROM strings;
```

`\d+` shows that the newly-typed table is only 1147 MB (vs the untyped table's 1438 MB). This is an improvement, but still a bit of a disappointment. Two things are hurting us: Postgres has a relatively high minimum per-row cost of about 40 bytes [^2], and representing a small number like `280` as a number rather than a string is actually more storage intensive (in exchange, query-time arithmetic will be faster).

Now our query on the unindexed-but-typed table takes only 550 ms (vs the unindexed-and-untyped table's 720 ms). This is because enumerated types are much cheaper to compare than strings.

Even better, creating an index on `geo_name` only takes 263 MB (vs 400 MB).

## SQLite

Postgres was very fast to slurp the data in and fast to query once we set up indexes, but it still resulted in a 1419 MB database. Can we do better?

### CSV virtual table module

SQLite supports exposing a CSV table in-place via the [CSV virtual table extension](https://www.sqlite.org/csv.html). Virtual tables are a lot like Postgres's foreign data wrappers.

I don't expect this will provide satisfactory query performance, but it's so easy to try that we may as well:

```
sqlite> .load ./csv.so
sqlite> CREATE VIRTUAL TABLE strings USING csv(filename='/home/cldellow/src/csv2parquet/census.csv');
sqlite> WITH inputs AS (
  SELECT
    c3 AS geo_name,
    CASE WHEN c10 = 1930 THEN 'total' ELSE 'cyclist' END AS mode,
    c14 AS female,
    c13 AS male
  FROM strings
  WHERE c10 IN ( 1930, 1935) AND
    c7 = 'CY' AND
    geo_name IN ('Victoria', 'Dawson Creek', 'Kitchener')
)
SELECT
  total.geo_name,
  cyclist.male,
  cyclist.female,
  100.0 * cyclist.male / total.male,
  100.0 * cyclist.female / total.female
FROM inputs AS total
JOIN inputs AS cyclist USING (geo_name)
WHERE total.mode = 'total' and cyclist.mode = 'cyclist';

Kitchener|905|280|1.51299841176962|0.514563998897363
Victoria|2650|2130|12.5741399762752|9.73047053449064
Dawson Creek|25|0|0.863557858376511|0.0
Run Time: real 24.721 user 23.944000 sys 0.776000
```

Yuck. The columns are identified by ordinal position (for example, `c3` instead of `geo_name`), and it's slow as molasses--almost half a minute.

### SQLite database

The main benefit of the virtual module was that it skipped the initial data loading step. Let's try a more traditional approach where we import the CSV into SQLite's database format.

```
cldellow@furfle:~$ sqlite3 census.sqlite
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
sqlite> .mode tabs
sqlite> .import census.tsv census
```

This creates a 1235 MB SQLite DB file. D'oh. And running our query against it takes 3.4 seconds! At a cost of 268 MB, we can add an index on `geo_name` that cuts query time down to 16 ms.

This is nicer than Postgres in that it's more self-contained and doesn't require us to administer a Postgres cluster, but it uses about the same amount of disk space.

### SQLite database, indexed + squashed

Our intuition is that the SQLite database is very compressible, since it still has all the repetitive data. Since this is for a read-only webapp, let's try using [SquashFS](https://en.wikipedia.org/wiki/SquashFS) to transparently compress it.

```
cldellow@furfle:~$ mkdir census
cldellow@furfle:~$ mv census.sqlite census
cldellow@furfle:~$ mksquashfs census/ census.squashfs -comp xz
[ ... buncha spam ... ]
cldellow@furfle:~$ mkdir /mnt/census
cldellow@furfle:~$ mount census.squashfs /mnt/dir -t squashfs -o loop
```

The compressed file is 185 MB. The first query takes about 130 ms, which is not bad! It's a bit slower because it has to uncompress the
necessary pages from the file. Subsequent queries are about 30 ms, because the pages are in the kernel buffer cache.

## Parquet

While easy to use, the previous solutions still use too much disk space! What if we explored Parquet files?

Parquet files partition your data into row groups which each contain some number of rows. Within those row groups, data is stored (and compressed!) by column, rather than by row. For example, if you had a dataset with 1,000 columns but only wanted to query the `Name` and `Salary` columns, Parquet files can efficiently ignore the other 998 columns. Row groups also know the minimum and maximum values for each column. If you had 10,000 row groups sorted by `Name` and only wanted to see entries for `Colin Dellow`, you could easily skip row groups whose statistics show that they don't contain any responsive entries.

One downside of Parquet files is that they're usually used in "big data" contexts. You need to have heavy-duty infrastructure like a Hive cluster to read them. Some recent work, like the [Apache Arrow](https://arrow.apache.org/) and [parquet-cpp](project://github.com/apache/parquet-cpp) projects, are changing this. I built a [SQLite virtual table extension for Parquet](https://github.com/cldellow/sqlite-parquet-vtable) on top of these libraries [^3].

### Stringly typed

Let's create a Parquet file from our CSV! To begin, we'll treat every column as a string. I created it with row groups of size 20,000, hoping that would allow us to efficiently query for a given city, since a city's 2,247 entries should be usually be entirely contained within a single row group. I used the brotli compression codec, which I found gave slightly better compression than Snappy while not sacrificing query speed.

It results in a 42 MB file. That's almost 3% the size of the CSV! But is it fast to query?

We'll change the query a bit -- the original one is optimized for Postgres's treatment of CTEs.

```
sqlite> .load parquet/libparquet.so
sqlite> CREATE VIRTUAL TABLE census USING parquet('./census.parquet');
sqlite> WITH total AS (
  SELECT
    geo_name,
    female,
    male
  FROM census
  WHERE
    geo_name in ('Dawson Creek', 'Victoria', 'Kitchener') AND
    csd_type_name = 'CY' AND
    profile_id = '1930'
), cyclist as (
  SELECT
    geo_name,
    female,
    male
  FROM census
  WHERE
    geo_name in ('Dawson Creek', 'Victoria', 'Kitchener') AND
    csd_type_name = 'CY' AND 
    profile_id = '1935'
)
SELECT
  total.geo_name,
  cyclist.male,
  cyclist.female,
  100.0 * cyclist.male / total.male,
  100.0 * cyclist.female / total.female
FROM total
JOIN cyclist USING (geo_name)

Dawson Creek|25|0|0.863557858376511|0.0
Kitchener|905|280|1.51299841176962|0.514563998897363
Victoria|2650|2130|12.5741399762752|9.73047053449064
Run Time: real 0.038 user 0.040000 sys 0.000000
```

40ms! Not as fast as Postgres with an index, but not bad considering how much less disk space it uses.

### Strongly typed

Because Parquet is so good at encoding and compressing repetitive values, creating a typed Parquet file isn't actually much of a savings: it decreases the file size by only 600 KB.

Still, it's useful to do, because it results in more meaningful statistics. For example, the string `"2"` is considered larger than the string `"10"`, because `2` comes after `1` in the alphabets computers use.

## Conclusion

<iframe id="datawrapper-chart-Yz2NI" src="//datawrapper.dwcdn.net/Yz2NI/1/" scrolling="no" frameborder="0" allowtransparency="true" style="width: 0; min-width: 100% !important;" height="279"></iframe><script type="text/javascript">if("undefined"==typeof window.datawrapper)window.datawrapper={};window.datawrapper["Yz2NI"]={},window.datawrapper["Yz2NI"].embedDeltas={"100":379,"200":304,"300":304,"400":279,"500":279,"700":279,"800":279,"900":279,"1000":279},window.datawrapper["Yz2NI"].iframe=document.getElementById("datawrapper-chart-Yz2NI"),window.datawrapper["Yz2NI"].iframe.style.height=window.datawrapper["Yz2NI"].embedDeltas[Math.min(1e3,Math.max(100*Math.floor(window.datawrapper["Yz2NI"].iframe.offsetWidth/100),100))]+"px",window.addEventListener("message",function(a){if("undefined"!=typeof a.data["datawrapper-height"])for(var b in a.data["datawrapper-height"])if("Yz2NI"==b)window.datawrapper["Yz2NI"].iframe.style.height=a.data["datawrapper-height"][b]+"px"});</script>

By using SQLite and Parquet files, you can achieve great compression without sacrificing query time _too_ much. You can also use it with xerial's excellent [JDBC connector for SQLite](https://github.com/xerial/sqlite-jdbc) if you'd like to expose it in a JVM application.

Give Parquet files and SQLite some thought the next time you're thinking about releasing a standalone webapp.

------
[^1]: Dataset 98-401-X2016055, if you want to dig it up yourself.

[^2]: See Erwin Brandstetter's excellent [Stack Overflow answer](https://stackoverflow.com/a/13570853/1011680) on this topic.

[^3]: Why SQLite? Remember that this is in the context of a hobby app running in a constrained environment. I explored a custom Presto connector that would let it read parquet files from the local file system, but didn't like the overhead requirements. I also considered writing a a [custom table function](https://db.apache.org/derby/docs/10.14/devguide/cdevspecialtabfuncs.html) for Apache Derby and a [user-defined table](http://www.h2database.com/html/features.html#pluggable_tables) for H2 DB. These had less overhead than Presto, but were architecturally much more primitive than both Presto and SQLite, and I was skeptical I could get good performance from them.
