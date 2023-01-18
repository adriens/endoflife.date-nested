# ❔ About

With this script, you'll be able, thanks to [`COPY`](https://www.postgresql.org/docs/current/sql-copy.html)
to load [`endoflife.date` PostgreSQL data](https://endoflife.date/postgresql) into an ease to use regular table. 

👉 You can play with this code on [Killercoda standard Ubuntu playground](https://killercoda.com/playgrounds/scenario/ubuntu).

# 🏁 Prerequisites

```shell
sudo apt-get update
sudo apt-get -y install postgresql-client jq httpie
```

# ⬇️ Get the data

Get the data as a `csv` file:

```clear
clear
http https://endoflife.date/api/postgresql.json |\
    jq -r '.[] | [.cycle, .eol, .latest, .latestReleaseDate, .lts, .releaseDate] | @csv' \
    > /tmp/psql-eol.csv
cat /tmp/psql-eol.csv
```

Then get and run a Postgres instance:

```shell
sudo docker pull postgres:15.1
sudo docker run --rm --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v /tmp:/tmp postgres:15.1
```

Then create and connect to the newly created database:

```shell
clear
export PGPASSWORD=docker
psql -h localhost -U postgres -d postgres -c "create database eol"
psql -h localhost -U postgres eol
```

Now, create the table:

```sql
CREATE TABLE psql_eol (
  cycle decimal,
  eol date,
  latest varchar(10) UNIQUE,
  latestReleaseDate date,
  lts boolean,
  releaseDate date,
  PRIMARY KEY (cycle)
);

COMMENT ON TABLE psql_eol IS 'This table contains PostgreSQL EoLs. See https://endoflife.date/postgresql for input data.';
```

Load the data:

```sql
COPY psql_eol FROM '/tmp/psql-eol.csv' DELIMITER ',' CSV ;
```

.. and check it has been loaded:

```sql
\! clear
select * from psql_eol;
```

# 🕹️ Play wth versions

🥁 Let's see the current version of our database :

```sql
SELECT version();
SELECT current_setting('server_version_num');
```

Add a dedicated column:

```sql
alter table psql_eol 
ADD COLUMN server_version_num integer;
```

... then feed it:

```sql
update psql_eol
set server_version_num = cast(cycle || lpad(substring(latest, 4,5),4,'0') as integer)
where cycle >= 10;

update psql_eol
set server_version_num = cast(substring(latest, 1,1) || lpad(substring(latest, 3,1), 2, '0') || lpad(substring(latest, 5,2), 2, '0') as integer)
where cycle < 10;
```

Finally enjoy the newly created `server_version_num` column:

```sql
select * from psql_eol;
```

## 🔗 Add some urls

First, add a dedicated column :

```sql
alter table psql_eol
add COLUMN release_url varchar;
```

Then feed it:

```sql
update psql_eol
set release_url = 'https://www.postgresql.org/docs/release/' || latest || '/'
where cycle > 10;

update psql_eol
set release_url = 'https://www.postgresql.org/docs/' || cycle || '/index.html'
where cycle <= 10;
```

Then enjoy the full table:

```sql
select * from psql_eol;
```

## 👮 Version check

Let's see if we currently are on the latest version of the current
cycle, therefore **we should have one line** 🤞:

```sql
select * from psql_eol
where
server_version_num = cast(current_setting('server_version_num') as integer);
```
