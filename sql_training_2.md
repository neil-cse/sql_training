# SQL Training

The last session was a while ago and we only spent a few minutes at the end looking at SQL, so I'm not going to assume any SQL knowledge.

SQL is generally used to interact with a database containing tables of data.

There are lots of different dialects of SQL, and which you can use depends on the database you're trying to talk to. As far as I know all of them share the same core keywords. We won't be looking at anything that differs between SQL dialect.

We will focus on reading, filtering and transforming data rather than creating or modifying it. These operations will be familiar to you from the R seminars (if you went to them), but the names of the commands are slightly different.

We'll use the Albion database as a source of example data. You can go to http://albion.r.cse.org.uk/pgweb/ to follow along.

pgweb is a good tool for exploring the data, but please don't use it for enormous queries or data extracts - you can use the 'extract results' page from Albion that we looked at last time - it will queue up requests so the server doesn't get overwhelmed and email you when they're ready to download.

## A basic command

```sql
SELECT * FROM aggregates.household LIMIT 1;
```

What's going on here?

`SELECT` says we are going to read some data, rather than `INSERT` or `UPDATE` or `CREATE TABLE`. All the commands we'll look at today will start with this.

`SELECT *` is a shortcut for saying 'select all the columns from this table'

`FROM aggregates.household` specifies which table we want to select from - `aggregates.household`. The table name is `household`; the schema is `aggregates`. A schema is just a group of tables (so `aggregates.building` also exists).

`aggregates.household` contains data on each household in England/Scotland/Wales, from Experian, EPCs, OS AddressBase, and a few other sources.

`LIMIT 1` is just there so it doesn't try and read all 27 million rows of household data - it means 'take the first row'. It's useful for previewing data. We haven't specified an order for the results to appear in, so it's just the first row that the database happens to stumble on. It isn't guaranteed to be the same one each time.

Let's try a few variations:

```sql
SELECT uprn, address, tenure_type FROM aggregates.household LIMIT 10;
```

```sql
SELECT uprn, postcode, heating_fuel FROM aggregates.household LIMIT 150;
```

## filtering results using 'WHERE'

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033';
```

This is the most basic form of `WHERE` - we only want to see values where the LSOA is E01016033. There's only 500 or so of these so we don't need the `LIMIT` any more.

You can get arbtrarily fancy in your combinations of filters using `AND`, `OR` and `NOT`:

```sql
SELECT uprn, postcode, heating_fuel 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033' AND (epc_band = 'C' OR tenure_type = 'Privately rented');
```

(careful with the brackets, though)

`IN`: easier than saying OR, OR, OR...

```sql
SELECT uprn, postcode, heating_fuel, tenure_type, epc_band, epc_score
FROM aggregates.household 
WHERE postcode IN ('ME7 1SX', 'ME7 2RL', 'ME7 2AL');
```

### SQL syntax

A few notes about SQL syntax:

This is invalid:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033'
WHERE epc_band = 'C';
```

Why? Because SQL is annoying. It needs to be written like this:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033' AND epc_band = 'C';
```

You can only have each of these clauses once. There are a few clauses you can repeat (mostly JOINs), but for the most part you can't.

This is invalid:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
LIMIT 100
WHERE lsoa_2011 = 'E01016033';
```

Why? Because SQL is annoying. It cares about the order of the clauses. The `LIMIT` has to be after the `WHERE`.

The error message won't help you out much, though... (show them the error message: `ERROR:  syntax error at or near "where"`)

This won't work:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
WHERE lsoa_2011 = "E01016033";
```

Why? at least in the dialect of SQL that the Albion database uses, double-quotes are reserved for quoting column or table names, which you might need to do if you somehow end up with a column or table name which isn't just made of lowercase characters/numbers and underscores. Single-quotes are for strings.

Also watch out for: missing commas, too many commas (show these)

You don't need to capitalise the clause names - sql is mostly case-insensitive, though how much that's true for things like table and column names varies by database. It's a fairly common style to capitalise the clause names, though, and can make it easier to read the query. For longer queries I also tend to put each clause on its own line.

### null

`NULL` in SQL can be a bit tricky. This query won't work:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type, organisation_name 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033' and organisation_name = null
LIMIT 100;
```

And neither will it's opposite (`organisation_name != null`). In theory, NULLs represent unknown values, and so they don't play nicely with other things. They don't even play nicely with each other:

Examples: `SELECT null = null;` `SELECT null != null;` `SELECT 1 != null`

What you can do instead is use `IS` and `IS NOT`: `organisation_name IS NOT null`

## ordering

ordering the results table is quite simple:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033'
ORDER BY postcode DESC;
```

`ORDER BY` specified which column (or columns) to order by. `ASC` or `DESC` specifies if it should be ascending or descending. Ascending is the default (a-z/0-9 rather than 9-0/z-a)

If you had a `LIMIT` clause too, it would go after the `ORDER BY`.

## groups and aggregating

Grouping:

```sql
SELECT 
    tenure_type, 
    count(*) AS households, 
    avg(num_adults) AS avg_adults, 
    min(epc_score) AS worst_epc_score, 
    max(epc_score) AS best_epc_score
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033'
GROUP BY tenure_type;
```

The `GROUP BY` clause specifies which column or columns to group by. It doesn't do much on it's own - you also need to say which aggregate functions you want to apply to the groups. For example:
* count the number of members in the group
* find the max or min value in a specified column for a group
* find the mean value in a specified column for a group
* lots of other things. See https://www.postgresql.org/docs/12/functions-aggregate.html

You can also see here using the `AS` keyword to rename a column in the output.

Grouping by multiple columns:

```sql
SELECT 
    tenure_type,
    heating_fuel, 
    count(*) AS households, 
    avg(num_adults) AS avg_adults, 
    min(epc_score) AS worst_epc_score, 
    max(epc_score) AS best_epc_score
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033'
GROUP BY tenure_type, heating_fuel
ORDER BY tenure_type;
```

What it won't let you do is SELECT a column which is not in the `GROUP BY` clause and doesn't have an aggregate function applied to it. Even if you the human knows that column `b` will always have the same value when you group by column `a`, the poor computer doesn't.

If you do need to do this, a way round it is just to use `MAX` or `MIN`.

## combining tables with 'JOIN'

The last major keyword to cover is `JOIN`

The JOIN keyword is how you combine data from multiple tables.

There's not a lot of need to join things on to `aggregates.household`, as that was the result of joining lots of things up in the first place, so let's look at some other tables.

Let's say you want to combine Experian data with the AddressBase classification, but `aggregates.household` didn't exist. You'd have to get both the Experian and EPC data:

```sql
SELECT hh.uprn, hh.tenure_type, addr.classification_code, addr.full_class_desc
FROM
    experian.household hh
    LEFT JOIN addressbase.address addr ON hh.uprn = addr.uprn
LIMIT 100
```

There's a few new things here.

`SELECT hh.uprn, hh.tenure_type, addr.classification_code, addr.full_class_desc` Looks the same as before, except with the 'hh.'s and 'addr.'s, which I'll talk about in a second.

`experian.household hh LEFT JOIN addressbase.address addr ON hh.uprn = addr.uprn`: whenever you do a join you also need to use the 'ON' keyword. This is how you tell the database how to join the 2 tables. Here I've also given short names to both tables (hh and a) to make referring to them easier. These are the same names being used to choose the columns. Then I'm saying 'for each row from Experian, find all the rows in the Addressbase data with the matching UPRN and combine them'.

Generally when joining, you will join `ON` the unique identifier of at least one of the tables. Here I'm using the UPRN, which is an AddressBase thing, and there's one for every address in the country.

### JOIN gotchas

Without the aliases it won't work, as both tables have a column called UPRN.:

```sql
SELECT uprn, tenure_type, classification_code, full_class_desc
FROM
    experian.household
    LEFT JOIN addressbase.address ON uprn = uprn
LIMIT 100
```

If multiple AddressBase addresses had the same UPRN (they don't) - you would get a row per duplicate, so you'd get some duplicate households. This can be OK but depends on your purposes.

If multiple Experian households and multiple AddressBase addresses have the same UPRN - you'd get a row per duplicate household for each row per duplicate address. (this is called a join explosion, because if there are lots of duplicates on both sides you can end up with enormous numbers of rows). This is rarely OK and can make things a) take ages and b) produce far too many rows

How to avoid these problems?
* Whoever made the database will should have specified some columns as unique or as primary keys. A primary key is a unique identifier of a database row. So that's always safe to join on. (show in data dictionary)
* postgres (which is the SQL database Albion uses) has the DISTINCT ON keyword. (I won't cover this now)

### Types of JOIN

We looked at `LEFT JOIN` above - what this means is that all the rows from the left-hand table will be included in the results, even if they don't match anything in the right-hand table.

You can also do an `INNER JOIN`: this only includes rows in the results if there's a match.

You can also do `RIGHT JOIN` and `FULL OUTER JOIN` but those aren't often needed, in practice.

If you want one of each row in the right-hand table for each row in the left, which is called the cross product, you can just put both tables in the `FROM` clause: `SELECT * FROM a, b`. This will produce a lot of results if your tables are even moderately large!

## Using SQL with non-Albion data

If you're working with large spreadsheets or CSVs in R, especially if you're transforming the data in time-consuming ways, you can save the results of intermediate steps to a local sqlite database

## SQL features I haven't covered

* subqueries
* windows
* functions
* working with spatial data using postGIS or spatialite
* lots more