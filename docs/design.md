# RedisGraph: A High Performance In-Memory Graph Database as a Redis Module


## Abstract

Graph based data is everywhere now days, Facebook, Google, Twitter and Pinterest are only a few who've realize the power behind relationship data and are utilizing it to the fullest, as a direct result we see a rise both in interest and variety of graph data solutions.

With the introduction of [Redis Modules](http://antirez.com/news/106) we've seen the great potential of introducing a graph data structure to Redis arsenal,
a native C implementation with emphasis on performance was developed to bring new graph database capabilities to Redis,
the [RedisGraph](http://redismodules.com/modules/redis-graph/) is now available as an open source project on [GitHub](https://github.com/swilly22/redis-module-graph).

In this document we'll discuss the internal design and feature of RegisGraph and demonstrate its current capabilities.


## RediGraph At-a-Glance

RediGraph is a graph database developed from scratch on top of Redis, using the new Redis Modules API to extend Redis with new commands and capabilities. Its main features include:
- Simple, fast indexing and querying
- Data stored in RAM, using memory-efficient custom data structures
- On disk persistence
- Tabular result sets
- Simple and popular query language (Cypher)
- Data Filtering, Aggregation and ordering


## A Little Taste: RediGraph in Action
Let’s look at some of the key concepts of RediGraph using this example over the redis-cli tool:

### Introducing our entities:
It is a common concept to represent entities as nodes within a graph, In this example, we'll create a small graph with both actors and movies as its entities,
an "act" relation will connect actors to movies they casted in.

We use the graph.CREATENODE command to create a new entity:
```sh
graph.CREATENODE <graph_id> <label> <attribute_name> <attribute_value> <attribute_name> <attribute_value> ...
```

Or in our example:
```sh
graph.CREATENODE IMDB actor name "Aldis Hodge" birth_year 1986
graph.CREATENODE IMDB actor name "OShea Jackson" birth_year 1991
graph.CREATENODE IMDB actor name "Corey Hawkins" birth_year 1988
graph.CREATENODE IMDB actor name "Neil Brown" birthyear 1980
graph.CREATENODE IMDB movie title "Straight Outta Compton" genre Biography votes 127258 rating 7.9 year 2015
graph.CREATENODE IMDB movie title "Never Go Back" gener Action votes 15821 rating 6.4 year 2016
```

### Connecting entities:
It is now time to form relationships between actors and movies, we use RedisGraph ADDEDGE command and specify the source entity, type of connection and destination entity as such:

```sh
GRAPH.ADDEDGE <graph_id> <src_entity_id> <relation> <dest_entity_id>
```

Adding the Straight Outta Compton cast:

```sh
GRAPH.ADDEDGE IMDB <Aldis_Hodge node id> act <Straight_Outta_Compton node id>
GRAPH.ADDEDGE IMDB <OShea_Jackson node id> act <Straight_Outta_Compton node id>
GRAPH.ADDEDGE IMDB <Corey_Hawkins node id> act <Straight_Outta_Compton node id>
GRAPH.ADDEDGE IMDB <Neil_Brown node id> act <Straight_Outta_Compton node id>
```

Adding the only cast member who also played in the movie Never Go Back.

```sh
GRAPH.ADDEDGE IMDB <Aldis_Hodge node id> act <Never_Go_Back node id>
```

### Querying the graph:
RedisGraph exposes a subset of Neo4J Cypher language, although only a number of language capabilities are supported there's enough functionality to extract valuable insights from your graphs, to execute a query we use the GRAPH.QUERY command:

```sh
GRAPH.QUERY <graph_id> <query>
```

Let's execute a number of queries against our movies graph:

Find the sum, max, min and avg age of the Straight Outta Compton cast:

```sh
GRAPH.QUERY IMDB "MATCH (a:actor)-[act]->(m:movie {title:\"Straight Outta Compton\"})
RETURN m.title, SUM(a.age), MAX(a.age), MIN(a.age), AVG(a.age)"
```

RedisGraph will reply with:

```sh
1) "m.title, SUM(a.age), MAX(a.age), MIN(a.age), AVG(a.age)"
2) "Straight Outta Compton,123.000000,37.000000,26.000000,30.750000"
3) "Query internal execution time: 0.071000 milliseconds"
```

The first row is our result-set hearder which name each column according to the return clause.
Second row contains our query result.
Last row contains RedisGraph execution time.

Let's try another query, this time we'll find in how many movies each actor played.

```sh
GRAPH.QUERY IMDB "MATCH (actor)-[act]->(movie) RETURN actor.name, COUNT(movie.title) AS movies_count ORDER BY movies_count DESC"

1) "actor.name, movies_count"
2) "Aldis Hodge,2.000000"
3) "O'Shea Jackson,1.000000"
4) "Corey Hawkins,1.000000"
5) "Neil Brown,1.000000"
6) "Query internal execution time: 0.071000 milliseconds"
```

## The Theory: Ideas behind RedisGraph

Different graph databases uses different structures for representing a graph, some use adjacency list others might use an adjacency matrix, each with its advantages and disadvantages, For RedisGraph it was crucial to find a data structure which will enable us to perform fast searches on the graph, and so we're using a concept called Hexastore to hold all relationships within a graph.

### Graph representation: Hexastore

A Hexastore is simply a list of triplets, where each triplet is composed of three parts:

1. Subject
2. Predicate
3. Object

Where the Subject refers to a source node, Predicate represents a relationship and the Object refers to a destination node.
For each relationship within a graph our hexastore will contain all six permutation of the source node, relationship edge and destination node, for example consider the following relation:

`(Aldis_Hodge)-[act]->(Straight_Outta_Compton)`

Aldis_Hodge is the source node
act is the relationship
and Straight_Outta_Compton is the destination node.

All six possibilities of representing this connection are as follows:

```
SPO:Aldis_Hodge:act:Straight_Outta_Compton
SOP:Aldis_Hodge:Straight_Outta_Compton:act
POS:act:Straight_Outta_Compton:Aldis_Hodge
PSO:act:Aldis_Hodge:Straight_Outta_Compton
OPS:Straight_Outta_Compton:act:Aldis_Hodge
OSP:Straight_Outta_Compton:Aldis_Hodge:act
```

With the Hexastore constructed we can easily search our graph, suppose I would like to find the cast of the movie Straight Outta Compton, all I've to do is search my Hexastore for all strings containing the prefix: `OPS:Straight_Outta_Compton:act:*`

Or if I'm interested in all the movies Aldis Hodge played in I can search for all strings containing the prefix: `SPO:Aldis_Hodge:act:*`

Although a Hexastore uses plenty of memory, six triplets for each relation, we're using a trie data structure which is not only fast in terms of search but is also memory efficient as it doesn't create duplication of string prefixes it already seen.

### Query language: Cypher
There are a number of Graph Query languages, we didn't want to reinvent the wheel and come up with our own language,
and so we've decided to implement a subset of one of the most popular graph query language out there Cypher by Neo4J,
the Open-Cypher project provides means to create a parser for the language, although convenient
we decided to create our own parser with Lex as a tokenizer and Lemon which generates a C target parser.

As mentioned only a subset of the language is supported, but it is our intention to continue adding new capabilities and extend the language.

## Runtime: query execution
Let's review the steps our module takes when executing a query,
consider the following query which finds all actors who've played alongside Aldis Hodge and are over 30 years old:
```
MATCH (aldis::actor {name:"Aldis Hodge"})-[act]->(m:movie)<-[act]-(a:actor) WHERE a.age > 30 RETURN m.title, a.name
```

RediGraph will
- Parse query, build abstract syntax tree (AST)
- Construct a query execution plan composed of:
  - Label scan operation
  - Filter operation (filter tree)
  - Expand operation
  - Expand into operation
- Execute plan
- Populate result-set with matching entities attributes

### Query parser
Given a valid query the parser will generate an AST containing four primary nodes one for each clause:

1. MATCH
2. WHERE
3. RETURN
4. ORDER

Generating an abstract syntax tree is a common way of discribing and structuring a language.

### Filter tree
A query can filter out entities by creating predicates, in our example we're filtering actors which are younger then 30.
It's possible to combined predicates using the OR, AND keywords to form granular conditions. during runtime the WHERE clause is used
to construct a filter tree, each node within the tree is either a condition e.g. A > B or an operation (AND/OR), candidate entities are passed
through the tree and get evaluated.


### Query processing
The MATCH clause describes relations between queried entities (nodes), a node can have an alias which will allow us to refer to it
at later stages within the executing query lifetime (WHERE, RETURN clause), but all nodes must eventually be assign an ID,
the process of assigning IDs to nodes is refer to as the search phase.

During the search we'll be querying the Hexastore for IDs according to the MATCH clause structure,
for instance in our example we'll start our search by looking for movies in which Aldis Hodge played in,
for each movie we'll extend our search to find out which other
actors played in the current processed movie.

As you might imagine the search process is a recursive operation which traverse the graph, at each step a new ID is
discovered, once every node has an ID assigned to it we can be assured that current entities have passed our filters,
at this point we can extract requested attributes (as specified in the return clause) and append a new record to the final result set.

## Benchmarks

Depending on the underlying hardware results may vary, that said inserting a new relationship is done in O(1) RedisGraph is able to create 100K new relations within one second.

Retrieving data really depends on the size of the graph and the type of query you're executing, on a small size graph ~1000 entities and ~2500 edges RedisGraph is able to perform ~65K friend of a friend query every second.

It's worth mentioning that besides the hexastore, entities are not indexed, it’s our intention to introduce entities indexing which should decrease query execution time dramatically.


## License
Redis-Graph is published under AGPL-3.0.

## Conclusion
RedisGraph although still a young project, can be an alternative to other graph databases, with its subset of operations one can use it to analyze and explore its graph data, being a Redis module this project is accessible from every Redis client without the need to make any adjustments. It's our intention to keep on improving and extending RedisGraph with the help of the open source community.
