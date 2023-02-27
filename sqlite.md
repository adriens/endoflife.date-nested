# ❔ About

With this script, you'll be able, thanks to [`.mode csv`](https://www.sqlitetutorial.net/sqlite-import-csv/)
to load [`endoflife.date` data](https://endoflife.date/) into an ease to use regular tables. 

👉 You can play with this code on [Killercoda standard Ubuntu playground](https://killercoda.com/playgrounds/scenario/ubuntu).

# 🏁 Prerequisites

```shell
sudo apt-get update
sudo apt install -y jq httpie sqlite3

```

# ⬇️ Get the data

Get the data as a `csv` files:

```shell
# Get products
# https://endoflife.date/api/all.json
cd /tmp

http https://endoflife.date/api/all.json |\
    jq -r '. []'\
    > _products.csv
wc -l _products.csv

# ⏳ read products and load eols
echo "product,cycle,release_date,eol,latest,link,lts,support,discontinued, extended_support" > eols.csv
while read -r product
    do
        # echo "Getting $product ..."
        url=https://endoflife.date/api/${product}.json
        echo " $url"
        # http  --ignore-stdin GET $url
        http  --ignore-stdin GET $url | jq -r '.[] | ["'"$product"'", .cycle, .releaseDate, .eol, .latest, .link, .lts, .support, .discontinued, .extendedSupport] | @csv' >> /tmp/eols.csv
    done < _products.csv

# Put some headers to csv
echo "products" > products.csv
cat _products.csv >> products.csv

git clone https://github.com/adriens/endoflife.date-nested.git
cat /tmp/endoflife.date-nested/data/categories.csv

```

# 🚀 Create tables and load `csv` datas

```shell
clear
cd

# Now create tables...
sqlite3 endoflife.date.sqlite

```

```sql

DROP TABLE IF EXISTS products; 
create table products(
    product text primary key
);

DROP TABLE IF EXISTS eols; 
create table eols(
    product text NOT NULL,
    cycle text NOT NULL,
    release_date text,
    eol text,
    latest text,
    link text,
    lts text,
    support text,
    discontinued text,
    extended_support text,
    CONSTRAINT PK_product_cycle PRIMARY KEY (product,cycle),
    FOREIGN KEY (product) REFERENCES products(product)
);

drop table if exists categories;
create table categories (
    id integer,
    category text primary key
);

drop table if exists product_categories;
create table product_categories(
    product text primary key,
    category text not null,
    FOREIGN KEY (product) REFERENCES products(product),
    FOREIGN KEY (category) REFERENCES categories(category)
);

```

```
.mode csv

```

```
.import --csv --skip 1 /tmp/products.csv products

```

```
.import --csv --skip 1 /tmp/eols.csv eols

```

```
.import --csv --skip 1 /tmp/endoflife.date-nested/data/categories.csv categories

```

```
.import --csv --skip 1 /tmp/endoflife.date-nested/data/product_categories.csv product_categories

```


```sql
-- Take a glance at datas
select * from eols limit 20;
select * from products limit 20;
select * from categories limit 20;
select * from product_categories limit 20;

-- See if we have eol for sqlite
select * from eols
where
    product='sqlite';

-- Add a metadata table
drop table if exists metadatas;
create table metadatas (
    key text not null PRIMARY KEY,
    value text not null
);

insert into metadatas values ("about", "This is an sqlite export made out of calls to endoflife.date's API (https://endoflife.date/docs/api)." );
insert into metadatas values ("www_about", "https://github.com/adriens/endoflife.date-nested/");
insert into metadatas values ("date_utc_publish", datetime('now'));
insert into metadatas values ("www_author_twitter", "https://twitter.com/rastadidi");
insert into metadatas values ("www_author_devto", "https://dev.to/adriens");
insert into metadatas values ("www_author_github", "https://github.com/adriens");
insert into metadatas values ("www_endoflife.date", "https://endoflife.date/");

select * from metadatas;

-- Compute and some datas to eols table to help better analysis

-- enrich eols.eol
alter table eols
    add eol_date text; 

update eols
    set eol_date = eol
where
    eol not in ("true", "false"); 

alter table eols
    add eol_boolean ineteger;

update eols
    set eol_boolean = 1
where
    eol = "true"; 

update eols
    set eol_boolean = 0
where
    eol = "false";

-- enrich eols.lts
alter table eols
    add lts_date text;

update eols
    set lts_date = lts
where
    lts not in ("true", "false"); 

alter table eols
    add lts_boolean integer;

update eols
    set lts_boolean = 1
where
    lts = "true";
    

update eols
    set lts_boolean = 0
where
    lts = "false";

-- enrich eols.support
alter table eols
    add support_date text;

update eols
    set support_date = support
where
    support not in ("true", "false"); 

alter table eols
    add support_boolean integer;

update eols
    set support_boolean = 1
where
    support = "true";
    
update eols
    set support_boolean = 0
where
    support = "false";

-- enrich eols.extended_suport
alter table eols
    add extended_support_date text;

update eols
    set extended_support_date = extended_support
where
    extended_support not in ("true", "false"); 

alter table eols
    add extended_support_boolean integer;

update eols
    set extended_support_boolean = 1
where
    extended_support = "true";
    

update eols
    set extended_support_boolean = 0
where
    extended_support = "false";

-- enrich with release_date_year
alter table eols
    add release_date_year text;

update eols
    set release_date_year = strftime('%Y', release_date); 

alter table eols
    add release_date_year_month text;


update eols
    set release_date_year_month = strftime('%Y-%m', release_date);
```


```
-- some reporting on metadas
.tables
.schema

.header on
.mode column
pragma table_info('eols');
pragma table_info('products');

-- self-test database before leaving session
.selftest
```

Finally exit `sqlite` : 

```shell
.exit
```

Now, let's have a look at the resulting file

```shell
# Check resulting database file
ls -ltr
file endoflife.date.sqlite
# prepare archives for later use
# tar cvzf endoflife.date.csv.tar.gz eols.csv products.csv
```

# 🤓 Fun with `litecli`

🔖 [`litecli`](https://litecli.com/features/)

```shell
# Install litecli
pip install -U litecli
```

```shell
# Open database
litecli endoflife.date.sqlite
```

😎 Now, just enjoy auto-completion... or simply run a firt report:

```sql
select category,
    count(*)
from product_categories
    group by category
    having count(*) > 10
    order by count(*) desc;
```

# 🗺️ Schema visualization

Now, let's [Visualize Your SQLite Database (with One Command)](https://dev.to/sualeh/how-to-visualize-your-sqlite-database-with-one-command-and-nothing-to-install-1f4m) : 

```shell
docker run \
--mount type=bind,source="$(pwd)",target=/home/schcrwlr \
--rm -it \
schemacrawler/schemacrawler \
/opt/schemacrawler/bin/schemacrawler.sh \
--server=sqlite \
--database=endoflife.date.sqlite \
--info-level=standard \
--command=schema \
--output-file=endoflife.date.png

file endoflife.date.png

```

Finally enjoy the resulting diagram:

```shell
xdg-open endoflife.date.png

```

# 🦆 Load into `DuckDB`

## About `DuckDB`

_"`DuckDB` is a column-oriented embeddable OLAP database"_ that has a lot of cool loading features.

Let's see what kind of cool things we can achieve with it around `sqLite`.

To install it, just:

```shell
brew install duckdb

```

Then:

``shell
duckdb --version
duckdb --help

```

Finally open the `DuckDB` shell:

```shell
duckdb endoflide.data.duckdb

```

## 🕹️ Playin' with 🦆 and 🪶

Load `sqlite` extension into `DuckDB`... and **mount its tables into `DuckDB`**:

```
INSTALL sqlite;
LOAD sqlite;
CALL sqlite_attach('endoflife.date.sqlite');

```

Now show `sqLite` tables from within `DuckDB`:

```
Show tables;
DESCRIBE eols;

```

## 📊 Make reports

Now, let's run queries in `DuckDB`:

```sql
SELECT category as "Cat.",
    count(*) "Count."
from product_categories
    group by category
order by count(*) desc;

```

```sql
select product, count(*)
from eols
group by product
order by count(*) desc
limit 20;

```

```sql
select product as "Product",
    count(*) as "Nb. cycles"
from eols
group by product
having count(*) > 15
order by count(*) desc;

```

Some reporting on mobile devices:

```sql
select product as "Product",
    count(*) as "Nb. Cycles",--cycle,
    max(release_date_year_month) as "Latest Release"
from eols
where
    product in ('nokia',
        'samsung-mobile',
        'iphone',
        'pixel',
        'fairphone')
group by product
order by count(*) desc;

```

## 🪶 Export to Apache `.parquet` file

> "Apache Parquet is an open source, column-oriented data file format designed for efficient
> data storage and retrieval. It provides efficient data compression and encoding schemes
> with enhanced performance to handle complex data in bulk."

S, just export the mounted table into a `.parquet` file : 

```
COPY eols TO 'endoflife.date_eols.parquet' (FORMAT PARQUET);
exit

```

Then check what the file looks like:

```shell
file endoflife_eols.parquet

```

Or take a look at datas with [`tad`](https://github.com/antonycourtney/tad):


```shell
tad endoflife_eols.parquet

```

Now, we can inspect it with [`parquet-tools`](https://github.com/ktrueda/parquet-tools) :

```shell
pip install parquet-tools

```

Finally inspect the `parquet` file:

```shell
parquet-tools inspect endoflife.date_eols.parquet
parquet-tools rowcount endoflife.date_eols.parquet
parquet-tools show endoflife.date_eols.parquet
```

We can also play with [`pqrs`](https://github.com/manojkarthick/pqrs) : 

```shell
brew install manojkarthick/tap/pqrs
```

Then:

```shell
pqrs --version
pqrs --help
```

```shell
pqrs cat data/cities.parquet
```
# 📊 Interact w. Kaggle

[Kaggle](https://www.kaggle.com/) has an [`API`](https://www.kaggle.com/docs/api), but
also a dedicated [`cli`](https://github.com/Kaggle/kaggle-api#installation) that makes it
very easy to interact (search & download datasets,...).

First install it:

```shell
pip install kaggle
```

... properly seup [API credentials](https://github.com/Kaggle/kaggle-api#api-credentials) and
you're ready:

```shell
kaggle datasets list -s endoflife
```

Finally have some fun around [`Kaggle`](https://www.kaggle.com/):

```shell
kaggle datasets list -s endoflife
```

```shell
kaggle datasets download adriensales/endoflifedate
file endoflifedate.zip
unzip endoflifedate.zip
ls -la endoflife.date.sqlite
file endoflife.date.sqlite
```
