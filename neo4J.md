# ❔ About

With this simmple code snippet, you'll be able to load Neo4J EoLs inside any
Neo4J instance.

# 🚀 Quickstart

```cypher
// Get the current version
call dbms.components() yield name,
  versions,
  edition
unwind versions as version
return name, version, edition;
```


```cypher
// Load all products
CALL apoc.load.json("https://endoflife.date/api/all.json")
  YIELD value
  UNWIND value.result AS product
MERGE (p:EOLProduct {name: product,
  url: 'https://endoflife.date/' + product});

// create unique index
CREATE CONSTRAINT EOL_UNIQUE_PRODUCTS
FOR (p:EOLProduct)
REQUIRE p.product IS UNIQUE;

MATCH (p:EOLProduct) RETURN p;


MATCH (p:EOLProduct) 
  CALL apoc.load.json('https://endoflife.date/api/neo4j.json')
  YIELD value
MERGE (c:EOLCycle {product: p.name,
       cycle: value.cycle,
       eol: value.eol,
       //support: value.support,
       latest: value.latest,
       latestReleaseDate: value.latestReleaseDate,
       releaseDate: value.releaseDate,
       lts: value.lts//link: value.link
       }
       );

// link cycles with products
MATCH
  (c:EOLCycle),
  (p:EOLProduct)
WHERE c.product = p.name
  CREATE (c)-[r:EOL_CYCLE_OF]->(p)
RETURN type(r);
```

... then get releases of Neo4J :

```cypher
MATCH (c:EOLCycle)-[r:EOL_CYCLE_OF]->(p:EOLProduct)
  WHERE p.name = 'neo4j'
RETURN c,r,p;
```

# 🔖 Related contents

- 📝 [🔁 Browse Neo4J EoL versions inside Neo4J AuraDB 🤓](https://dev.to/optnc/browse-neo4j-eol-versions-inside-neo4j-auradb-1ncb)
- 🎥 [Video demo](https://youtu.be/rpRDstHf-fk)
