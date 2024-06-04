## Chapter-2: Data Models And Query Languages

How we structure our data models has a profound effect on how we write software and also on how we think about the problem. All the applications are built by layering one data model on top of another. For instance, object data structure models in our programming languages are built upon json/xml/tables/graph models that are used by storage systems, which in turn is built upon bytes in memory, which in turn is build upon the electrical signals captured in the hardware.

Data models provide different ways for storing and querying of data. The most widely used data models are the relational model, the document model and graph-based model. 

*Relational Models:*
In relational models the data is organized into relations/tables, where each relation is an unordered collection of tuples/rows. Relational models impose schema-on-write(where the schema is explicit and ensures all the data written conforms to it). Each entity in our application will be modelled as a seperate relation and any attributes within the entity that could take multiple values will be moved into another relation(i.e., in case of a User entity, we will have a user relation that has the attributes like name, email, phone which are single valued attributes. But user's education, which is a multi valued attribute will be moved into a new relation called users_education and a link will be created between them to find particular user's education details). This link is referred as foreign key. This kind of relationship is referred to as one-to-many relationship. Relational models provide joins using the foreign key to gather the scattered attributes of an entity and represent the whole data. This aspect was considered as a drawback in case of applications where all attributes of the entity are queried at once, but the relational model imposed a constraint of splitting the attributes even though it was not an application requirement, hence caused overheads like multiple joins and an awkward tranlation layer required between the objects in the application code and the database model of tables, rows and columns.
Querying on relational models in done using SQL(Structured Query Language). SQL is a declarative query language [in declarative language, we just specify the pattern, conditions and transformations needed in our end result, but not the execution order to achieve the result. In contrast, for imperative languages we tell the computer to perform certain operations in a certain order]. Based on the details provided in our sql query, the database system's query optimizer decides which indexes to use and in which order to execute various parts of the query to get the result. To some extend we can workaround this drawback by duplicating attributes across multiple relations, then the schema is not normalized and will need application code to maintain consistency with updates to such attributes. Some of the relational model based databases are Mysql, postgresql.

*Document models:*
Document models take advantage of the data locality in cases where the entire entity is needed at once. Document models are schemaless, which is actually schema on-read(where the structure of the data is implicit and only interpreted when the data is read). It uses JSON format to store all the attributes of an entity as a single document. Attributes having multiple values will be nested as sub-documents. If our application has data in a document-like structure(i.e., a tree of one-to-many relationships, where typically the entire tree is loaded at once), then document models are better fit. Document models better suit usecases having a simple one-to-many relationships. It is also recommended to keep the document size fairly small for better retrieval and also the documents should not be deeply nested, otherwise referring to a nested item within a document becomes complex. Document models are not suitable for many-to-many relationships and many-to-one relationships as it doesn't have good support to joins.
Document models provide JSON based query language for data retrieval. Also mongodb introduced aggregation pipeline which is a declarative query language. Some of the document model based databases are MongoDB, CouchDB, RethinkDB.

*Graph models:*
Graph-based models treat data as vertices connect by edges. Each vertex donotes an entity and edges represents the relationship between the entities. The vertices and edges of graph can have homogeneous or heterogeneous data. Graph models are best suited for data having complex many-to-many relationships where anything is potentially related to everything. Graph models provide two ways of structuring and querying data which are property graph model and triple-store model.
In property graph,   
i. each vertex consists of 
  - a unique identifier
  - a set of outgoing edges 
  - a set of incoming edges 
  - a collection of properties(key-value pairs)  
ii. each edge consists of  
  - a unique identifier
  - the vertex at which the edge starts(tail vertex)
  - the vertex at which the edge ends(head vertex)
  - a label to describe the kind of relationship between the two vertices
  - a collection of properties(key-value pairs)  
Cypher is a declarative query language for property graph model, created for Neo4j graph database.

In triple-store model, all the information is stored in form of very simple three-part statements:(subject, predicate, object). The subject is equivalent to a vertex in a graph. The object can be a primitive data type, in which case the predicate and subject form the key-value property(bottle, colour, black) or another vertex in the graph, in which case the predicate is an edge in the graph, the subject is the tail vertex and the object is the head vertex(lucy, marriedTo, alan). This triple format is also called turtle. 
SPARQL is a query language for triple-store models.

```
The key ideas are,
* relational database serve well for less complex many-to-many relationships.
* document databases target usecases where data comes in self-contained documents and relationships between one document and another is rare or the relationship is just one-to-many.
* graph database target usecases with complex many-to-many relationship where anything is potentially related to everything.
```
