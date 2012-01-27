---
title:  Developing a Search API
layout:  post
---

# Introduction #

In order to attain maximum flexibility in our user interface, we must design a flexible search API internally.  It should support all major search operators discussed in this document.

Traditionally, the following operators are available:

 * `AND`
 * `OR`
 * `NOT`

When dealing with Google, these would be utilized in the following query:

    "csci 1000u" OR "csci 2020u" -1030u

These operators can be ambiguous at times.  In order to deal with this, many systems introduce a method of disambiguation.  This allows the user to clarify their intent.

For example,

    csci AND 1000u OR 2020u

But one may wish to place emphasis on the `OR` first.

    csci AND (1000u OR 2020u)

# Internal Search API #

Transferring a human-friendly representation into a data structure may prove slightly challenging.  It would require parsing the query in order to correctly interpret it.

## Simple Queries ##

A simple query is defined as one that does not utilize manual disambiguation.



# Topics For Discussion #

 * Specify analysis of term(s) - eg. N-gram, whitespace, etc.
 * Wildcards
 * How to handle stop words - Are they part of the query or filler?
