---
layout:  post
title:  Encoding of a Relational Database into Full-Text Search Database
---
## Introduction ##

Performing full-text search on a relational database can be an expensive operation.  This is typically  a linear operation.  In order to allow for nearly instantaneous search speeds this data must be encoded into a full-text search database.  In the case of this document Lucene will be used.

## Crawling the Relational Database ##
While the database schema can be inferred from the database itself, allowing manual configuration allows for maximum control.  You may ignore some columns, for instance.

### Entity Configuration ###

An entity configuration entry is made up of the following components:

 * `Type` - A type can be either an `:value`, `:entity`, or `:group`
 * `Class` - The class represents the kind of entity it is
 * `SQL` - The SQL query required to produce the entities (currently implemented with ClojureQL)
 * `ID` - The attribute column (or columns) that uniquely identify each entity
 * `Attrs` - A listing of attributes for each entity
 * `Values` - Defines which columns to be added to the values index from each entity class

For example,

<figure>
{% highlight clj %}
(EntitySchema.
   {:T :entity
    :C :courses
    :sql (table :courses)
    :ID :code
    :attrs [:code :title :description]
    :values [:title]})
{% endhighlight %}
	<figcaption>Example entity schema configuration.</figcaption>
</figure>

### Crawling Procedure ###

Each entity configuration is capable of crawling the database for its own entities.  In order to do this, the point of entry to the application must pass along the database connection, full-text search database connection, and a writer for the full-text database.

Since it knows the SQL required to extract all of its entities, it takes it and executes a query against the database.  For every result returned, each row is encoded into a document and added to the FTDB.

#### Handling Values ####

One SQL query would be enough to handle extract all of the values from the database, but it would not extract all _unique_ values.  In order to do this, the query is modified for every value column specified in the entity configuration file.  The column values are projected out and grouped together.  By doing this all unique values for each column are found.  These are then encoded into a document as well and added to the FTDB.

## Encoding Values, Entities, and Entity Groups ##

Each row returned by the database query is in the form of a map.  For example,

<figure>
{% highlight clj %}
{:code "csci 5300g" :description "Networks." :title "Computer Communication"}
{% endhighlight %}
	<figcaption>Sample database row in Clojure data structure form.</figcaption>
</figure>

Essentially a row contains all attributes specified in the entity configuration for that particular entity.  This is enough in a relational database as additional information can be stored in places such as the table name and primary keys, but not in a full-text database.  In a full-text database there is no concept of a fixed schema so this additional information must be stored in another form.

### Data Structure Representation ###

Internally in Clojure each row is stored as a special map.  Each map consists of the requested attributes, as well as additional meta-data.  This meta-data contains additional information such as the type, class, and identifier for the particular entity.

An example of this would be as follows,

<figure>
{% highlight clj %}
(def attrs
  {:code "csci 5300g" :description "Networks." :title "Computer Communication"})

(with-meta attrs
           {:__type__ :entity
            :__class__ :course
            :__id__ "course|csci_5300g"})
{% endhighlight %}
	<figcaption>Internal Clojure entity representation.</figcaption>
</figure>

By encoding entities in this manner, we are able to preserve important information without polluting the data structure itself with what should be considered meta-data.

The in-memory data structure entity representation is used internally to allow for flexibility.  Both rows and documents can be encoded or decoded using this format.  This allows internal code to be able to handle entities from either a relational database or search results from the full-text search database.  By utilizing an in-memory format we reduce code complexity as well as create more flexibility.

### Encoding Data Structure ###

In order to support the encoding and decoding from different sources, we utilize multi-methods.  This allows us to dispatch the correct encoding or decoding function based on the type given.  This works as Lucene Documents are of type `Document` and database rows are of type `IPersistentMap`.

#### From Database Row ####

Each database row returned contains a superset of the attribute columns specified by the entity configuration.  In addition, it also includes a unique identifier used by the database.

For the internal data structure, this gives the attributes as well as part of the `:__id__` field.  The problem is that some information specified in the database is missing.  This information comes from the entity definition.

The entity definition provides the missing `:type`, `:class`, and other part of the unique identifier.  It also specifies which attributes are of interest in the case that the database schema holds additional information that is not to be indexed.  This is easily accomplished with the `select-keys` function.

For example,

<figure>
{% highlight clj %}
(select-keys row attrs)
{% endhighlight %}
	<figcaption>Selecting required attributes from a row.</figcaption>
</figure>

Thus each entity data structure instance is composed of data from the following sources:

 * Entity schema definition
 * The database row itself

By composing together the two sources of information, a complete entity can be created.

#### From Existing Document ####

When converting back to the intermediate representation from a Lucene document, we can make certain assumptions.  Each document is stored with meta-data prefixed and suffixed with two underscores.  In addition, only attributes of interest are stored in each document.  Thus encoding a document is as simple as pulling out required meta-data and disassociating the meta-data from the attributes.
