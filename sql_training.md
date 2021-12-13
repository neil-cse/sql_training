# SQL

Structured Query Language

A very old programming language

SQL is generally used to interact with a database containing tables of data.

We will focus on reading, filtering and transforming data rather than creating or modifying it. These operations will be familiar to you from the R seminars (if you went to them), but the names of the commands are slightly different.

We'll use the Albion database as a source of example data. You can go to albion.r.cse.org.uk/pgweb/ to follow along.

pgweb is a good tool for exploring the data, but please don't use it for enormous queries or data extracts - there's a different tool for that, which I'll show later.

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
SELECT uprn, postcode, heating_fuel, tenure_type FROM aggregates.household WHERE lsoa_2011 = 'E01016033';
```

This is the most basic form of `WHERE` - we only want to see values where the LSOA is E01016033. There's only 500 or so of these so we don't need the `LIMIT` any more.

You can get arbtrarily fancy in your combinations of filters using `AND`, `OR` and `NOT`:

```sql
SELECT uprn, postcode, heating_fuel FROM aggregates.household WHERE lsoa_2011 = 'E01016033' AND (epc_score = 'C' OR tenure_type = 'Privately rented');
```

(careful with the brackets, though)

### indexes

The reason these queries are fast, even with over 20 million rows to look through, is because there's an index on the column `lsoa_2011`. This means the database doesn't need to look through all the rows; it just consults the index to find all of the rows with the specified LSOA.

You can still filter on columns without an index, but on big tables (which is most of them in Albion), it'll be slow.

How to find out if a column has an index? albion.r.cse.org.uk/data-dictionary . Also possible to see in pgweb.

## ordering

TODO

## combining tables with 'JOIN'

The JOIN keyword is how you combine data from multiple tables.

There's not a lot of need to join things on to `aggregates.household`, as that was the result of joining lots of things up in the first place, so let's look at some other tables.

Let's say you want to combine Experian data with the AddressBase classification, but `aggregates.household` didn't exist. You'd have to go the Experian and EPC data:

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

Something to watch out for with joins:
* What happens if multiple AddressBase addresses have the same UPRN? (they don't) - you would get a row per duplicate, so you'd get some duplicate households.
* What happens if multiple Experian households and multiple AddressBase addresses have the same UPRN? - you'd get a row per duplicate household for each row per duplicate address. (this is called a join explosion, because if there are lots of duplicates on both sides you can end up with enormous numbers of rows)

How to avoid this?
* Whoever made the database will often have specified some columns as primary keys. A primary key is a unique identifier of a database row. So that's always safe to join on.
* postgres (which is the SQL database Albion uses) has the DISTINCT ON keyword. (I won't cover this now)

### types of join

In the above example I used a LEFT JOIN. There are quite a few other types of join, but it's unlikely you'll need any of them except INNER JOIN. 

## groups and aggregating

TODO

## data dictionary

TODO

## Albion models

TODO

## results extractor

TODO
