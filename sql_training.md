# Albion

Just jump in with any questions as we're going - it's hard to pitch technical content at exactly the right level for the audience, so let me know if I'm assuming you know something that you don't.

What is Albion?

Albion is an internal tool developed within research at CSE. You can find at albion.r.cse.org.uk (need to be on VPN or in the office).
It brings together datasets about households, addresses and buildings in the UK, processes them to be easier to use (cleaning up free text fields and so on, address-matching), and combines them together.

On top of this, several models have been made that use the Albion datasets to do things like:
* predict the solar PV potential of rooftops
* predict heat demand for buildings
* predict how easy it will be to dig up a road

So there are 2 mains ways you can use Albion: running models, and extracting data. You can also combine the two.

My plan for today is to:
* first run through how to use the models; 
* then through the different datasets that are available in Albion; 
* and then look at some SQL. You do need to have a bit of SQL knowledge to make the most of the data, though not for just running the models. But even then, knowing some SQL would all you to combine the model outputs with some of the datasets, which could be useful.

## Albion models

***go through models***

http://albion.r.cse.org.uk/run-jobs

## Albion datasets

To make use of the datasets and the processing that's been done on them, you really need some SQL knowledge. I'll go through all the datasets available first, though.

Some examples of datasets:
* OS MasterMap - footprint shape and height of all buildings in the UK, layout of road network
* OS AddressBase - all addresses in the UK
* EPCs - heating systems and fuel types, recommended insulation, energy efficiency, floor area ...
* Experian - facts about households (age of house, number of people, number of children, tenure type ...)
* Experian Mosaic - market segmentation of households
* average electricity and gas usage at postcode/LSOA/MSOA level
* if a building is grade I/II listed
* presence of PV
* OA/LSOA/MSOA/LA/ward/parish of all addresses

***go through datasets***

http://albion.r.cse.org.uk/data-dictionary

# SQL

Structured Query Language

A very old programming language

SQL is generally used to interact with a database containing tables of data.

There are lots of different dialects of SQL, and which you can use depends on the database you're trying to talk to. As far as I know all of them share the same core keywords. We won't be looking at anything that differs between SQL dialect.

We will focus on reading, filtering and transforming data rather than creating or modifying it. These operations will be familiar to you from the R seminars (if you went to them), but the names of the commands are slightly different.

We'll use the Albion database as a source of example data. You can go to http://albion.r.cse.org.uk/pgweb/ to follow along.

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

`IN`: faster than saying OR, OR, OR...

```sql
SELECT uprn, postcode, heating_fuel, tenure_type, epc_band, epc_score
FROM aggregates.household 
WHERE postcode IN ('ME7 1SX', 'ME7 2RL', 'ME7 2AL');
```

### indexes

The reason these queries are fast, even with over 20 million rows to look through, is because there's an index on the column `lsoa_2011`. This means the database doesn't need to look through all the rows; it just consults the index to find all of the rows with the specified LSOA.

You can still filter on columns without an index, but on big tables (which is most of them in Albion), it'll be slow - maybe about 5 minutes.

How to find out if a column has an index? albion.r.cse.org.uk/data-dictionary . Also possible to see in pgweb.

## ordering

ordering the results table is quite simple:

```sql
SELECT uprn, postcode, heating_fuel, tenure_type 
FROM aggregates.household 
WHERE lsoa_2011 = 'E01016033'
ORDER BY postcode DESC;
```

`ORDER BY` specified which column (or columns) to order by. `ASC` or `DESC` specifies if it should be ascending or descending. Ascending is the default (a-z/0-9 rather than 9-0/z-a)

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

## combining tables with 'JOIN'

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

Something to watch out for with joins:
* What happens if multiple AddressBase addresses have the same UPRN? (they don't) - you would get a row per duplicate, so you'd get some duplicate households.
* What happens if multiple Experian households and multiple AddressBase addresses have the same UPRN? - you'd get a row per duplicate household for each row per duplicate address. (this is called a join explosion, because if there are lots of duplicates on both sides you can end up with enormous numbers of rows)

How to avoid this?
* Whoever made the database will often have specified some columns as primary keys. A primary key is a unique identifier of a database row. So that's always safe to join on.
* postgres (which is the SQL database Albion uses) has the DISTINCT ON keyword. (I won't cover this now)

### things I haven't covered

* types of join - e.g. the difference between `LEFT JOIN` and `INNER JOIN`
* subqueries
* windows
* functions
* lots more

## painful things

* SQL error messages are not very good and often don't indicate precisely where the mistake in the query is. So if it's telling you that the query is wrong, but the position it's indicating seems fine, look closely at the bits before and after the position it tells you.
* SQL doesn't allow trailing commas in lists - `SELECT a,b,c, FROM blah` will fail due to the comma after the c.
* in my experience, it takes a while to learn how to express what you want in SQL for anything more complicated than what we've covered.
* combining the different keywords isn't always straightforward. How would you join 2 tables, but perform a grouping on one of them before joining?

## results extractor

Now you've got your query, how do you actually get any data out?

***show results extractor***

http://albion.r.cse.org.uk/extract-results
