---
jupytext:
  formats: ipynb,md:myst
  main_language: sql
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.1
kernelspec:
  display_name: Bash
  language: bash
  name: bash
---

# UN VOTES DATABASE

+++

Author and maintainer: Vadim Rudakov, rudakow.wadim@gmail.com

The “UN votes” Database is a collection of normalized data of the UN voted resolutions’ details from 1946 to present, collected from the [UN Digital Library](https://digitallibrary.un.org/search?c=Voting+Data&cc=Voting+Data&ln=en). The database is intended as a source for machine learning experiments with international relations data, such as building predictive models for future votes or clustering countries. That is why the main goal of the database is the countries’ vote results. Not all the resolutions presented have this information, but we decided to include the details on all available resolutions for those researchers who would want to conduct other type of analysis where the vote results do not play significant role.

The database is updated monthly to incorporate newly voted resolutions. See [RELEASE_NOTES.md](RELEASE_NOTES.md) for the current version statistics.

+++

# Quick Start

+++

Download the latest `un_votes.sql.gz` from [Releases](https://github.com/soviar-systems/un_votes/releases).

+++

## Using Podman (recommended)

+++

No local PostgreSQL installation required — everything runs inside the container. The database is automatically created and populated on first start:

``` bash
# Start PostgreSQL and restore the dump (one command)
podman run -d --name un-votes-postgres \
  -e POSTGRES_DB=un_votes \
  -e POSTGRES_USER=user1 \
  -e POSTGRES_PASSWORD=12345 \
  -v ./un_votes.sql.gz:/docker-entrypoint-initdb.d/un_votes.sql.gz:Z \
  docker.io/library/postgres:17

# Connect to the database
podman exec -it un-votes-postgres psql -U user1 -d un_votes
```

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.resolution
 LIMIT 5;"
```

The first start takes about a minute while PostgreSQL processes the dump. To stop and remove the container when done: `podman rm -f un-votes-postgres`

> **How it works:** the official PostgreSQL container image automatically executes any `.sql`, `.sql.gz`, or `.sh` files found in the `/docker-entrypoint-initdb.d/` directory on first startup. By mounting our dump there, the database is created and populated without any manual steps. This only happens once — subsequent container starts skip the initialization if data already exists.

+++

## Using an existing PostgreSQL cluster (advanced)

+++

If you already have a running PostgreSQL server and know how to administer it:

``` bash
# Create a user and database (run as a PostgreSQL superuser, e.g. postgres)
psql -U postgres -c "CREATE USER user1 WITH PASSWORD '12345';"
psql -U postgres -c "CREATE DATABASE un_votes WITH OWNER user1;"

# Restore the dump
gunzip -c un_votes.sql.gz | psql -U user1 -d un_votes

# Connect
psql -U user1 -d un_votes
```

+++

## Verify the installation

+++

Run the built-in summary function:

``` sql
SELECT * FROM un.get_database_statistics();
```

We will show the real examples here using the direct connection to the database in the container:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.get_database_statistics();"
```

# Part I. How the database was collected and built

+++

This document covers the technical details of how the database was collected and how you can work with it to get the most out of it.

The entire project of building the database can be divided into 3 big parts.

1.  Analysis of the data provided by the UN Digital Library and the database architecture elaboration. We had to decide:

- what kind of data was accessible to us and what kind of data should have really been stored for our purposes,
- how to organize this data to make it easy 1) to update it with the new data, and 2) to work with for analysts,
- what naming and data types we should choose for the database schemata.

“[Architecture](#Architecture)” gives answers on how we solved this problems.

1.  Next step involved gathering, processing, and sending the data from the UN Digital Library website to the PostgreSQL database.

For this task we wrote the crawler using the popular Python’s “scrapy” module. For sending the data to the database we used another Python’s module “psycopg 3”. The processing pipeline was written in a way to treat each scrapy’s item (i.e. all the details of one resolution) as a transaction, within which all the operations of inserting data and getting the foreign keys for already inserted data either completed successfully or aborted completely. This allowed the maintainer to be sure that all the data was consistent and keep track of bad transactions to rewrite the code during the development and test stage.

1.  And the last part was to maintain the system of logs.

This step was essential to have a right to release the database to the community - we could not share the data we did not trust ourselves. Logs system was to control the correctness of two processes: 1) sending the data to the database (client side) and 2) storing the data in the database (server side), and make us be able to trace any kind of corrupted data during the database development step to make changes in the source code. We do not open our source code to prevent misuse that could overload the UN Digital Library servers, but we share all the logs so everyone can see the entire way of the data from the website to the database. All the scraping and sending data to the database job was automated, no hand work was implemented at all to exclude any kind of human made mistake.

+++

# Part II. Working with the database

+++

## Architecture

+++

To get the full out of the database, one should understand its architecture. If you have worked with the relational databases before and have the familiarity with the `JOIN` operation, you will quickly grasp the idea of how to work with the database. The maintainers intended to implement the classic “many-to-many” model.

The architecture idea of the database comes from two sources:
- the resolution’s available details,
- the strive for the database normalization, i.e. the principle “one string in one place” realization.

This is the example of the typical webpage with the resolution details:

:::{figure} ./images/Screenshot_20240311_185702.png

Figure 1. UN Resolution page on the UN Digital Library website with the details
:::

On the Fig. 2 you can see the attributes that have been processed to the database:

:::{figure} ./images/Screenshot_20240311_185702_1.png

Figure 2. Attributes that came into the database on the UN Digital Library website with the details
:::

As you may have noticed, no information on “Draft”, “Note” and “Vote summary” has been deemed valuable for our purposes. These data contain no additional useful information but would have overloaded the database and slowed down its performance unnecessarily. Regarding the “Vote summary” field, this information can be easily derived from the `vote` table where we store all the votes country by country, so this information should not take its own storage space. Probably, the only reason we should somehow incorporate this information is that “NON-RECORDED” resolutions, i.e. resolutions without country-by-country votes, cannot be identified as accepted or rejected within our database. If there is a real need for such information, we will rewrite the crawler and rebuild the database, but for now this information is not included.

All the highlighted fields keep their names in the database, so you can easily switch between the web-page and the query result when needed.

:::{note} Why the crawler filters with `fct__9=Vote`

The UN Digital Library’s [Voting Data collection](https://digitallibrary.un.org/search?c=Voting+Data&cc=Voting+Data&ln=en) contains ~23,500 records in total, but only ~10,000 of them are typed as “Vote”. The remaining ~13,500 are resolutions adopted without vote (by consensus). The crawler uses the `fct__9=Vote` search facet to collect only the records where a voting procedure took place.

The UN uses three adoption methods:

1.  **Recorded vote** — country-by-country roll-call, our primary data.
2.  **Non-recorded vote** — show of hands; we know the totals but no per-country breakdown. These are in the database with 0 rows in the `vote` table.
3.  **Adopted without vote** — consensus, no voting procedure at all. These are the ~13,500 excluded records.

The `fct__9=Vote` facet captures categories 1 and 2, which is exactly the right boundary for a vote analysis database. Category 3 records have no vote data whatsoever — including them would add thousands of rows to the `resolution` table with no analytical value for vote pattern research.
:::

This is the database final architecture:

:::{figure} ./images/architecture.png

Figure 3. UN Votes Database architecture
:::

+++

## Tables

+++

Now let’s take a look at the tables and their interaction with each other.

There are 10 tables in the database that contain the UN resolutions data, and also there is an additional bilingual table with the general info about the database (`readme`, filtered by `lang` column: `'en'` or `'ru'`). This is the list of the the UN data containg tables:

1.  `agenda`
2.  `committee_report`
3.  `country`
4.  `meeting_record`
5.  `resolution`
6.  `resolution_agenda`
7.  `resolution_committee_report`
8.  `title`
9.  `vote`
10. `vote_choice`

You can see the entire list of the tables using command `\dt`, and also you can use `\d` to see the schema of each table, for example:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\dt un.*"
```

## “resolution” table

+++

The `resolution` table is the core table - its `resolution.id` attribute binds all the tables together. But if you compare the resolution attributes on the page in the Fig. 1 and the names of tables in the tables’ list you will notice that some of these attributes have their own tables and some don’t. The `resolution` table has only 5 columns:
- `id` - record number that you can use to quickly find the web page by substituting the “RECORD” word in the url `https://digitallibrary.un.org/record/RECORD?ln=en` with this id (for example, the resolution from the Figure 1 has `id = 278340` and its web address is, then, “https://digitallibrary.un.org/record/278340?ln=en”)
- `title_id`,
- `symbol` is the UN documentation standard, see https://research.un.org/en/docs/symbols for details,
- `meeting_record_id`,
- `vote_date`.

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.resolution"
```

### How resolution’s attributes are stored

+++

The `resolution` table has only a few attributes because some resolutions’ attributes may have more than one distinct value. There are only three such attributes: `agenda`, `committee_report`, and `vote`, all of them have gotten their own tables for data integrity purposes (see details how to use them in later sections).

Consider example for the resolution with id `518324`. It has ten (!) values for agenda:

:::{figure} ./images/Screenshot_20260427_204309.jpeg

Figure 5. Eight values for [agenda](https://digitallibrary.un.org/record/518324?ln=en)
:::

Here’s the record in our database, no agenda column:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.resolution 
WHERE id = 518324;"
```

We send all 8 agenda values to the `agenda` table where each new agenda value gets its unique id:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.agenda 
WHERE name ~* 'S/59.*65.*croatia situation.*';"
```

Then we update an intermediary table called `resolution_agenda` that have two columns - `resolution_id` (the foreign key to `resolution.id`) and `agenda_id` (the foreign key to `agenda.id`).

Thus, we get all the agendas for the given resolution in the intermediate table `resolution_agenda`:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.resolution_agenda 
WHERE resolution_id = 518324;"
```

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT a.name AS agenda_name
FROM un.resolution_agenda ra
JOIN un.agenda a ON ra.agenda_id = a.id
JOIN un.resolution r ON ra.resolution_id = r.id
JOIN un.title t ON r.title_id = t.id
WHERE ra.resolution_id = 518324;"
```

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.agenda WHERE id = 9327
LIMIT 5;"
```

Here you can see the top 10 resolutions with the largest number of `agenda` values:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- Top 10 resolutions with the most agenda items
SET search_path TO un;
SELECT
    resolution_id,
    count(resolution_id) AS cnt   -- how many agenda items this resolution has
FROM resolution_agenda
GROUP BY resolution_id
ORDER BY cnt DESC
LIMIT 10;"
```

and the same rating for `committee_report`:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- Top 10 resolutions with the most committee reports
SET search_path TO un;
SELECT
    resolution_id,
    count(resolution_id) AS cnt   -- how many committee reports this resolution has
FROM resolution_committee_report
GROUP BY resolution_id
ORDER BY cnt DESC
LIMIT 10;"
```

### JOIN all the attributes

+++

Now we can `JOIN` these tables to get the the original information (you can also change the format view to resemble the website format adding `\gx` to the end of the command in terminal):

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SET search_path TO un;
-- All GA resolutions with full details (no votes)
SELECT
    r.id AS record,               -- resolution ID, also matches UN Digital Library record ID
    t.name AS title,              -- resolution title
    a.name AS agenda,             -- agenda item text
    r.symbol AS resolution,       -- UN document symbol (A/RES/...)
    mr.symbol AS meeting_record,  -- meeting record symbol
    cr.symbol AS committee_report,-- committee report symbol
    r.vote_date                   -- date the vote took place
FROM resolution r
JOIN title t ON r.title_id = t.id
JOIN resolution_agenda ra ON r.id = ra.resolution_id
JOIN agenda a ON ra.agenda_id = a.id
JOIN meeting_record mr ON r.meeting_record_id = mr.id
JOIN resolution_committee_report rc ON r.id = rc.resolution_id
JOIN committee_report cr ON rc.committee_report_id = cr.id
WHERE r.symbol ~ '^A'             -- only General Assembly resolutions
ORDER BY r.vote_date DESC
LIMIT 5;"
```

This command returned all the attributes for the resolutions voted in the General Assembly, except for the vote, for each resolution in the descending order.

And this command will return the same attributes but also the vote of the Egypt:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SET search_path TO un;
-- All GA resolutions with Egypt's vote in each
SELECT
    r.id AS record,
    t.name AS title,
    a.name AS agenda,
    r.symbol AS resolution,
    mr.symbol AS meeting_record,
    cr.symbol AS committee_report,
    r.vote_date,
    vc.choice AS egypt_vote       -- resolved vote_choice_id to 'yes'/'no'/'abstentions'/'non-voting'
FROM resolution r
JOIN title t ON r.title_id = t.id
JOIN resolution_agenda ra ON r.id = ra.resolution_id
JOIN agenda a ON ra.agenda_id = a.id
JOIN meeting_record mr ON r.meeting_record_id = mr.id
JOIN resolution_committee_report rc ON r.id = rc.resolution_id
JOIN committee_report cr ON rc.committee_report_id = cr.id
JOIN vote v ON r.id = v.resolution_id
JOIN vote_choice vc ON v.vote_choice_id = vc.id
WHERE r.symbol ~ '^A'
  AND v.country_id = (            -- subquery: find Egypt's country_id by name (case-insensitive)
      SELECT id FROM country WHERE name ~* '.*egypt.*'
  )
ORDER BY r.vote_date DESC
LIMIT 5;"
```

If you’re struggling in understanding this syntax, please, consider learning SQL basic commands and relational databases design models (for example: “[PostgreSQL for Everybody](https://www.coursera.org/specializations/postgresql-for-everybody)”).

+++

## “agenda” table

+++

The `agenda` table contains two columns:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.agenda"
```

Figure 1 shows no special `subject` field but if you carefully look at the `agenda` string for the resolutions adopted after 1983, you will notice that the last part of the string is often capitalized:

:::{figure} ./images/Screenshot_20240323_185659.png

Figure 8. Agenda strings newer than 1983 examples
:::

The capitalized portion is a **subject** that describes the `agenda` in a more general manner. Previously, we extracted this into a separate `subject` table, but the extraction was unreliable (the UN does not use a consistent format — early resolutions have no subject at all, and Security Council agenda strings follow entirely different patterns). Since the subject is derivable from the agenda string itself, maintaining it as a separate table was a normalization violation with no practical benefit.

> **Working with Subjects:** If you need subject-level filtering, you can perform pattern-based filtering on `agenda.name` using SQL. Subjects are often embedded within the `agenda.name` field (typically after a double dash `--`). Since the format is inconsistent, we recommend using `LIKE` or `ILIKE` patterns for specific categories (e.g., `WHERE name ILIKE '%--%DISARMAMENT%'`). For official UN subject taxonomies, please refer to the [UN Digital Library search page](https://digitallibrary.un.org/search?cc=Voting%20Data&ln=en&p=&f=&rm=&sf=&so=d&rg=50&c=Voting%20Data&c=&of=hb&fti=0&fti=0).

+++

## “committee_report” table

+++

This table contains two columns:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.committee_report"
```

One resolution may have:
- zero,
- one,
- many

`committee_report` values.

That is why the additional - `resolution_committee_report` - table was created, designed for binding `resolution.id`s and their corresponding `committee_report.id` values:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.resolution_committee_report LIMIT 10;"
```

## “country” table

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.country"
```

The crawler processes data year by year starting from 1946 to present, so countries appear in the `country` table roughly in the order they first voted in the UN bodies (General Assembly and Security Council). This can be used as a rough indicator of when a country joined the UN, but should not be taken as the ultimate truth — the order depends on which resolution within a year the country first appears in.

## “meeting_record” table

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.meeting_record"
```

This table contains two columns:
- `id`, and
- `symbol` (UN Documentation standard).

There can only be one `meeting_record` value for the resolution, but many resolutions may share the same `meeting_record`.

## “title” table

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.title"
```

There can only be one `title` value for the resolution, but many resolutions may share the same `title`.

## “vote” table

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.vote"
```

This table has been the main purpose of the database - each contry’s vote results that you can use for interesting analyses.

This table contains only foreign keys:
- `resolution_id`,
- `country_id`,
- `vote_choice_id`

that you can use to get the vote of the country you are interested in for the given resolution.

## “vote_choice” table

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\d un.vote_choice"
```

This additional table contains only 4 rows - one row for each possible vote choice: ‘yes’, ‘no’, ‘abstentions’, and ‘non-voting’. If you look at the Fig. 1 example, you will see that the `vote` field uses empty string for non-voting countries and ‘Y’, ‘N’ and ‘A’ for other variants. We processed these values, changed them to the numbers on our choice, and created this table with the explanation of the vote result:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"SELECT * FROM un.vote_choice;"
```

If you have any questions or suggestions, feel free to contact the maintainer.

# Part III. SQL queries examples

Use `\pset format wrapped` command to get the lines in console wrapped for the better view experience.

## Database summary information

See [Quick Start](#quick-start) for usage of `un.get_database_statistics()` and a sample query.

## “JOIN” all details, except for “vote”

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- All resolutions with full details (both GA and SC, no votes)
SET search_path TO un;
SELECT
    r.id AS record,
    t.name AS title,
    a.name AS agenda,
    r.symbol AS resolution,
    mr.symbol AS meeting_record,
    cr.symbol AS committee_report,
    r.vote_date
FROM resolution r
JOIN title t ON r.title_id = t.id
JOIN resolution_agenda ra ON r.id = ra.resolution_id
JOIN agenda a ON ra.agenda_id = a.id
JOIN meeting_record mr ON r.meeting_record_id = mr.id
JOIN resolution_committee_report rc ON r.id = rc.resolution_id
JOIN committee_report cr ON rc.committee_report_id = cr.id
LIMIT 5;"
```

## “JOIN” all details only for GA, except for “vote”

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- Same as above, filtered to General Assembly resolutions only
SET search_path TO un;
SELECT
    r.id AS record,
    t.name AS title,
    a.name AS agenda,
    r.symbol AS resolution,
    mr.symbol AS meeting_record,
    cr.symbol AS committee_report,
    r.vote_date
FROM resolution r
JOIN title t ON r.title_id = t.id
JOIN resolution_agenda ra ON r.id = ra.resolution_id
JOIN agenda a ON ra.agenda_id = a.id
JOIN meeting_record mr ON r.meeting_record_id = mr.id
JOIN resolution_committee_report rc ON r.id = rc.resolution_id
JOIN committee_report cr ON rc.committee_report_id = cr.id
WHERE r.symbol ~ '^A'            -- GA resolutions start with 'A'
LIMIT 5;"
```

## “JOIN” “vote” results of one “country” for each resolution

For example Egypt, in descending order by date

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- All GA resolutions with Egypt's vote
SET search_path TO un;
SELECT
    r.id AS record,
    t.name AS title,
    a.name AS agenda,
    r.symbol AS resolution,
    mr.symbol AS meeting_record,
    cr.symbol AS committee_report,
    r.vote_date,
    vc.choice AS egypt_vote       -- 'yes', 'no', 'abstentions', or 'non-voting'
FROM resolution r
JOIN title t ON r.title_id = t.id
JOIN resolution_agenda ra ON r.id = ra.resolution_id
JOIN agenda a ON ra.agenda_id = a.id
JOIN meeting_record mr ON r.meeting_record_id = mr.id
JOIN resolution_committee_report rc ON r.id = rc.resolution_id
JOIN committee_report cr ON rc.committee_report_id = cr.id
JOIN vote v ON r.id = v.resolution_id
JOIN vote_choice vc ON v.vote_choice_id = vc.id
WHERE r.symbol ~ '^A'
  AND v.country_id = (
      SELECT id FROM country WHERE name ~* '.*egypt.*'
  )
ORDER BY r.vote_date DESC
LIMIT 5;"
```

## Get the records from a specific year

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- All resolutions voted in 2024
SELECT * FROM un.resolution
WHERE EXTRACT(YEAR FROM vote_date) = 2024
LIMIT 5;"
```

## Count the number of values in a resolution’s field

For example, let’s count the number of `committee_report` values for each resolution in descending order:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- How many committee reports each resolution has
SELECT
    resolution_id,
    COUNT(*) AS cnt               -- number of committee reports
FROM un.resolution_committee_report
GROUP BY resolution_id
ORDER BY cnt DESC
LIMIT 5;"
```

and filter only those resolutions that contain more than 2 `committee_report` values:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- Resolutions with more than 2 committee reports
SELECT
    resolution_id,
    COUNT(*) AS cnt
FROM un.resolution_committee_report
GROUP BY resolution_id
HAVING COUNT(*) > 2               -- filter: keep only those with 3+ reports
ORDER BY cnt DESC
LIMIT 5;"
```

## Show “agenda” from a particular year

Let’s see `agenda` in the General Assembly resolutions since 1983:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- Agenda items for GA resolutions from 1983 onward
SET search_path TO un;
SELECT
    r.id AS resolution,
    a.name AS agenda,
    r.vote_date
FROM agenda a
JOIN resolution_agenda ra ON a.id = ra.agenda_id
JOIN resolution r ON r.id = ra.resolution_id
WHERE EXTRACT(YEAR FROM r.vote_date) >= 1983
  AND r.symbol ~* '^a'            -- General Assembly only
ORDER BY r.vote_date
LIMIT 5;"
```

and for the Security Council since 1985:

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"-- Agenda items for SC resolutions from 1985 onward
SET search_path TO un;
SELECT
    r.id AS resolution,
    a.name AS agenda,
    r.vote_date
FROM agenda a
JOIN resolution_agenda ra ON a.id = ra.agenda_id
JOIN resolution r ON r.id = ra.resolution_id
WHERE EXTRACT(YEAR FROM r.vote_date) >= 1985
  AND r.symbol ~* '^s'            -- Security Council only
ORDER BY r.vote_date
LIMIT 5;"
```

## Save the result in csv

For example, save the `agenda` topics by year (extracting `subject` from agenda strings):

+++

``` sql
-- Export subjects by year to CSV
-- Note: only works for agenda strings that use the standardized format
\copy (
    SELECT DISTINCT ON (a.name)
        substring(a.name FROM '^\S+\s+\d+[a-z]?\s+([^:]+):') AS subject,  -- extract text between item number and colon
        EXTRACT(YEAR FROM r.vote_date) AS year
    FROM agenda a
    JOIN resolution_agenda ra ON a.id = ra.agenda_id
    JOIN resolution r ON r.id = ra.resolution_id
    WHERE a.name ~ '^\S+\s+\d+[a-z]?\s+[^:]+:'    -- only strings with extractable subject
    ORDER BY a.name, year
) TO '~/UN_Analysis/subjects.csv' WITH CSV HEADER;
```

+++

> **Note:** The standardized agenda format (`A/660 32 Subject : subtitle`) became common only from 1983 onward. Earlier resolutions do not have extractable subjects in this format — the regex filter `WHERE a.name ~ '...'` in the query above excludes them automatically.

# Misc

## Indexes

Two B-Tree expression indexes are created automatically by the pipeline after each crawl:

+++

``` sql
CREATE INDEX IF NOT EXISTS year_b ON resolution(EXTRACT(YEAR FROM vote_date));
CREATE INDEX IF NOT EXISTS month_b ON resolution(EXTRACT(MONTH FROM vote_date));
```

+++

These speed up queries that filter by year or month (e.g. `WHERE EXTRACT(YEAR FROM vote_date) = 2024`). A plain B-tree on `vote_date` would not help such queries — PostgreSQL cannot use a column index to satisfy an expression predicate.

```{code-cell}
podman exec un-votes-postgres psql -U user1 -d un_votes -c \
"\di un.*"  | grep -E 'year_b|month_b'
```

<!-- #endregion -->
