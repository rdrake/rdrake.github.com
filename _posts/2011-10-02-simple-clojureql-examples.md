---
title: Simple ClojureQL Examples
layout: post
---
## Dependencies ##

At the very least, you must include the ClojureQL and a driver for your database as a dependency.  In my case, this is Sqlite.  In addition you also need the Clojure JDBC library.

    [sqlitejdbc "0.5.6"]

    [org.clojars.mccraigmccraig/clojureql "1.1.0-SNAPSHOT"]

## Examples ##

### Required Namespaces ###

{% highlight clj %}
(ns examples.clojureql
    (:require [clojure.java.jdbc :as sql]
        [clojureql.core :as cql]))
{% endhighlight %}

### Database Information ###

{% highlight clj %}
(def db {
    :classname "org.sqlite.JDBC"
    :subprotocol "sqlite"
    :subname "path/to/db.sqlite3"})
{% endhighlight %}

### Relational Algebra ###

{% highlight clj %}
(def sel (-> (cql/table :tbl)
    (cql/project [:c1 :c2 ...])))
{% endhighlight %}

Can also be read as:

$$ \pi_{c_1, c_2, ...}(tbl) $$