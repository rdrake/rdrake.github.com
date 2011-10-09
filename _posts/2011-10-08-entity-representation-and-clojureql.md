---
layout: post
title:  Entity Representation and ClojureQL
---
## Tuples to ClojureQL ##

In the database, all rows are represented as tuples.  Given the following schema:

$$ R(c_1, c_2, ...) $$

And a tuple with values:

$$ (v_1, v_2, ...) $$

ClojureQL transforms these tuples into a map in the following form:

$$ \\{ :c_1 v_1 :c_2 :v_2 ... \\} $$

## Entity Representation ##

Entities require a few pieces of information:

 * Type or class
 * Unique identifier
 * Attributes

Based on the ClojureQL map structure above, we can infer the attributes of an entity.  Unfortunately entities cannot be represented by their attributes alone.  We need the type and a unique idenfier as well.  This problem is solved via an entity definition.

### Entity Definition ###

Every entity type used must be defined in a configuration file.  Each definition specifies the following information:

 * Entity type
 * Relational algebra necessary to produce said entity
 * Values to index

For example:

{% highlight clj %}
(def entities
  ;^{:private true}
  {:course
     {:name "Course"
      :sql (->
             (cql/table :courses)
             (cql/project [[:code :as :id] :title :description]))
      :values [:code :title :desc]})
{% endhighlight %}

By combining the information provided by the entity definition, as well as the attributes retrieved by ClojureQL, we can easily transform a ClojureQL map into an entity.

$$ f(ent-def, row) \rightarrow entity $$

The question becomes how should an entity be represeted as a data structure in Clojure.  The simplest method would be via a map.

{% highlight clj %}
{:__type__ "Course"
 :__id__ "csci 4100u"
 :__attrib__
    {:code "csci 4100u"
     :title "Mobile Devices"
     :description "This course is an introduction to..."}}
{% endhighlight %}

Some map entries are considered "special" in that they provide metadata rather than actual attributes of an entity.

## Entity to Document ##

The next step would be transforming these entities into documents in a full-text search index.  These documents are essentially just fields (key, value) that can either be stored, indexed, or tokenized (S.I.T.).  In the case of our data structure above for an entity, we would want all of the data structure fields in the document.

Unfortunately the full-text search engine of choice, Lucene, is only capable of searching one field at a time.  Thus if we place all attributes in separate fields, we would have to issue a query on the index for every possibly field.  This presents a need for an indexed, tokenized field that contains all attributes concatenated together.

{% highlight clj %}
(use '[clojure.string :only (join)])

(def entity
  {:__type__ "Course"
   :__id__ "csci 4100u"
   :__attrib__
    {:code "csci 4100u"
     :title "Mobile Devices"
     :description "This course is an introduction to..."}})

(def full-entity
  (into entity (join " " (vals (entity :__attrib__)))))
{% endhighlight %}

While tempting to just store the original attributes, it's possible for the user to want to be able to use a prefix in order to search on a specific attribute.  They could, for instance, want to search for all computer science courses.

    __type__:Course +code:csci

Thus all attributes must be indexed, tokenized, and stemmed, while the first two special fields should be indexed and whitespaced analyzed, but not stemmed.

### Document Metadata ###

In order to specify the S.I.T. parameters for a field in a document, we will make use of metadata.  In Clojure, this is easy to add:

{% highlight clj %}
(def field {:code "csci 4100u"})

(def field-with-meta
  (with-meta field
    { :store false :index true :tokenize false}))
{% endhighlight %}

To be continued...
