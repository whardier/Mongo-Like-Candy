=============================
Cascaded Multi-Clause Queries
=============================

:Author: Shane R. Spencer <shane@bogomip.com>
:Date: Wed Aug 28 02:25:34 UTC 2013

.. contents ::
    :backlinks: entry

Preface
=======

** WORK IN PROGRESS **

MongoDB_ definitely encourages developers to think out of the box and 
create clever query solutions that don't completely rely on the 
database server to do all the processing. One clever solution I refer 
to as "Cascaded Multi-Clause Queries" focuses on combining various 
index regions through a mixture of client side and server side 
logic.

"Cascading" refers to organizing each step of a multi-clause index so 
that certain index regions are processed in a specific order.  Usage 
of the limit_ cursor method allows for the query to exit without 
processing all the clauses if the limit size is reached.

MongoDB_ supports multi-clause queries by way of the or_ query 
operator and can be seen as cursor.explain.clauses_.

This query solution is incredibly useful for a variety of highly 
specific queries and has a very unique pagination property.

* Selecting one index area before another.

* Finding geographically aware documents without 2d_ or 2dsphere_ 
  indexes.

* Creating an index pyramid where smaller focused areas are queried 
  before larger areas.

* Pseudo-sorting very large volumes of data without requiring using 
  cursor.sort_.

* Removing index ranges from pagination results as the result set 
  grows.

Summary
=======

I inadvertantly found out about multi-clause queries examining a well 
crafted query that I was hopeful would do what I wanted.  I was using 
or_ on different values of a location field.  The goal for me was to 
first query documents with a specific location and then all the 
surrounding cities in an order I had determined before making the 
query.

To take advantage of this I knew I would need to feed the query an 
ordered set of locations which I could pregenerate or potentially let 
the user define.

Data
====

This article focuses on using a sample stream of geolocated Twitter_ 
posts using the `Twitter Streaming API`_.  It may not surprise any of 
you that the JSON_ output from Twitter_ can be directly imported into 
MongoDB_ using mongoimport_ and contains valid GeoJSON_ for direct use 
with the 2dsphere_ special index.

Twitter_ posts make an **excellent** data source to use when testing 
indexing requirements like multi-lingual text searching, geospatial 
data, multi-key indexes, and works very well when you simply need a 
lot of very unique data to play with.

.. topic :: **Example Post** (simplified)

  .. code :: javascript
        
        {
            "_id" : ObjectId("521d3eb8e5dee42bee224700"),
            "created_at" : "Wed Aug 28 00:02:55 +0000 2013",
            "text" : "not really sure how to feel about this",
            "user" : {
                "screen_name" : "some_dude",
                "geo_enabled" : true,
            },
            "coordinates" : {
                "type" : "Point",
                "coordinates" : [
                    -87.8333797,
                    41.50161718
                ]
            },
            "place" : {
                "name" : "Frankfort",
            }
        }







..  code-block :: javascript

    db.tweets.find({
        '$or': [{
            'place.name': 'Los Angeles'
        }, {
            'place.name': 'Manhattan'
        }, {
            'place.name': 'Houston'
        }]
    }).explain(verbose = true)

Now we see that there are several clauses, each with their own plans, 
that have describe each of the values I am searching for in the query.

.. code-block :: javascript

    {
        "clauses": [{
            "cursor": "BtreeCursor place.name_1",
            //...
            "indexBounds": {
                "place.name": [
                    [
                        "Los Angeles",
                        "Los Angeles"
                    ]
                ]
            }
        }, {
            "cursor": "BtreeCursor place.name_1",
            //...
            "indexBounds": {
                "place.name": [
                    [
                        "Manhattan",
                        "Manhattan"
                    ]
                ]
            }
        }, {
            "cursor": "BtreeCursor place.name_1",
            //...
            "indexBounds": {
                "place.name": [
                    [
                        "Houston",
                        "Houston"
                    ]
                ]
            }
        }],
    }

The Problem
===========

Geolocated Twitter_ posts contain field and location information 
through the ``places`` field which focuses on the geographic name of 
the city or state and the ``coordinates`` field that defines where on 
Earth the post was approximately made.


Combining a text command_ with other queries is somewhat difficult and 
nearly always requires the use of a two-stage query using an interim 
table when searching large collections of data.

As an example I would like to search for the word ``"feel"`` starting 
with the city ``"Frankfort"`` which has it's own coordinates as well.

We have a few approaches we can take.

1.  Search ``place.name`` for ``"Frankfort"`` as part of a a text_
    search command and then rerun the query for the next city we want 
    to search.

    ..  code :: javascript 
    
        db.tweets.runCommand("text", {
            search: "feel",
            filter: {
                "place.name": "Frankfort"
            }
        })

        db.tweets.runCommand("text", {
            search: "feel",
            filter: {
                "place.name": "Chicago"
            }
        })

    Doing this can result in a very long list of queries and can make 
    paginating through results troublesome.  We will also have to have 
    a compound index that isolates common language information into 
    different areas of the index based on the ``place.name`` 
    associated with the document.    

2.  Store the results of a geospatial query in an interim collection
    that has a text search_ index enabled as a simple index or as a 
    compound index following a unique search identifier like a new 
    ObjectId_.

3.  Similarly, store the results of a text command_ in an interim 
    collection that has some geospatial indexing enabled.  Depending 
    on how frequently a word is used in your ``post`` collection you 
    may see some very large 
    





It is a very good idea to use TTL_ indexes on interim collections so 
that staged data is eventually removed.

The documents in a very large collection I was working with all had a 
specific location on earth defined as a latitude and longitude.  They 
also had textual information I wanted to search for.  The goal was to 
search for textual content starting at a specific location on earth.

This is actually a simple matter with MongoDB if you use 

The data I was working with was a bunch of points on earth that are 
assigned a `Geohash <http://en.wikipedia.org/wiki/Geohash>`_ that 
describes the latitude and longitude of the point.  For instance a 
very precise Geohash of the centroid of Anchorage, Alaska is 
``bdvkkbmvn39b`` and for Wasilla, Alaska we'd be looking at 
``bdvwr6t98ejh``.  Both of those share a common prefix of ``bdv`` 
which is a very large area of earth.  See `Geohash Explorer 
<http://geohash.gofreerange.com/>`_ to examine your town and how large 
each Geohash level is.

Each document shared some textual information that I was going to use 
along with the MongoDB Text Search and I wanted to make sure I was 
only doing text searches based around a specific area with the 
documents closest to the area first.

Sharding
--------

The combination of Geospatial indexes and Text Indexes on top of a 
sharded collection was just not going to work with MongoDB the way I 
wanted it to.  In order to utilize a `2dsphere 
<http://docs.mongodb.org/manual/core/2dsphere/>`_ or `2d 
<http://docs.mongodb.org/manual/core/2d/>`_ index on top of a sharded 
cluster you need another field to act as the shard key.  In this 
situation I opted to use a Geohash as the shard key since it relates 
to the actual location of each document within the collection and can 
work in tandem with, or as a replacement for, the ``2dsphere`` index I 
was planning on using initially.
    
Pagination
----------

The Solution
============

Serialized Ocument Notation (SON)
=================================

The actual order of queries is VERY important and it is highly 
recommended you migrate your code to use the SON objects

#.. code-block:: javascript

Manual Geohashes?
=================

Even though it's not the primary focus of this article I wanted to 
quickly say why I was using geohashes in my own way.

References
==========

.. target-notes::

..  _or: http://docs.mongodb.org/manual/reference/operator/or/

..  _2d: http://docs.mongodb.org/manual/core/2d/

..  _2dsphere: http://docs.mongodb.org/manual/core/2dsphere/

..  _limit: http://docs.mongodb.org/manual/reference/method/cursor.limit/

..  _cursor.explain: http://docs.mongodb.org/manual/reference/method/cursor.explain/

..  _cursor.sort: http://docs.mongodb.org/manual/reference/method/cursor.sort/

..  _cursor.explain.clauses: http://docs.mongodb.org/manual/reference/method/cursor.explain/#or-query-output-fields

..  _mongoimport: http://docs.mongodb.org/manual/reference/program/mongoimport/

..  _GeoJSON: http://docs.mongodb.org/manual/reference/glossary/#term-geojson

..  _twitter: http://twitter.com/

..  _twitter streaming api: https://dev.twitter.com/docs/streaming-apis

..  _text search: http://docs.mongodb.org/manual/core/text-search/

..  _text command: http://docs.mongodb.org/manual/reference/command/text/

..  _objectid: http://docs.mongodb.org/manual/reference/object-id/

..  _mongodb: http://www.mongodb.org/

..  _aggregate: http://docs.mongodb.org/manual/reference/command/aggregate/

..  _ttl: http://docs.mongodb.org/manual/tutorial/expire-data/

..  _geohash: http://en.wikipedia.org/wiki/Geohash
