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

# read products and load eols
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
cat /tmp/endoflife.date-nested/data/categories.csv`

```

# 🚀 Create tables and load datas

```shell
clear
cd

# Now create tables...
sqlite3 endoflife.date.sqlite

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
    extended_support text
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
    category text,
    FOREIGN KEY (product) REFERENCES products(product),
    FOREIGN KEY (category) REFERENCES categories(category)
);

-- Load data from csv files
.mode csv
.import --csv --skip 1 /tmp/products.csv products
.import --csv --skip 1 /tmp/eols.csv eols
.import --csv --skip 1 /tmp/endoflife.date-nested/data/categories.csv categories
.import --csv --skip 1 /tmp/endoflife.date-nested/data/product_categories.csv product_categories

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
create table metadatas(
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
    add eol_boolean;

update eols
    set eol_boolean = eol
where
    eol in ("true", "false"); 

-- enrich eols.lts
alter table eols
    add lts_date;

update eols
    set lts_date = lts
where
    lts not in ("true", "false"); 

alter table eols
    add lts_boolean;

update eols
    set lts_boolean = lts
where
    lts in ("true", "false");

-- enrich eols.support
alter table eols
    add support_date;

update eols
    set support_date = support
where
    support not in ("true", "false"); 

alter table eols
    add support_boolean;

update eols
    set support_boolean = support
where
    support in ("true", "false");

-- enrich eols.extended_suport
alter table eols
    add extended_support_date;

update eols
    set extended_support_date = extended_support
where
    extended_support not in ("true", "false"); 

alter table eols
    add extended_support_boolean;

update eols
    set extended_support_boolean = extended_support
where
    extended_support in ("true", "false");

-- enrich with release_date_year
alter table eols
    add release_date_year;

update eols
    set release_date_year = strftime('%Y', release_date); 

alter table eols
    add release_date_year_month;


update eols
    set release_date_year_month = strftime('%Y-%m', release_date);
    
-- some reporting on metadas
.tables
.schema

.header on
.mode column
pragma table_info('eols');
pragma table_info('products');

-- self-test database before leaving session
.selftest

-- quit
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

😎 Now, just enjoy auto-completion.

# 📊 Interact w. Kaggle

- https://github.com/kaggle/kaggle-api
- https://www.kaggle.com/
