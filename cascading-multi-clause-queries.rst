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

This article focuses on using a sample stream of geotagged Twitter_ 
posts using the `Twitter Streaming API`_.  It may not surprise any of 
you that the JSON_ output from Twitter_ can be directly imported into 
MongoDB_ using mongoimport_ and contains valid GeoJSON_ for direct use 
with the 2dsphere_ special index as well as an array of points that 
works well with the 2d_ special index.

Twitter_ posts make an **excellent** data source to use when testing 
indexing requirements like multi-lingual text searching, geospatial 
data, multi-key indexes, and works very well when you simply need a 
lot of very unique data to play with.

Sample Document
---------------

Geotagged Twitter_ posts contain location information through the 
``places`` field which focuses on the geocoded city or state 
information for the ``coordinates`` field that defines where on earth 
the post was approximately made from.

..  code :: javascript

    // Simplified
    
    {
        "_id" : ObjectId("521e8e89aca6a342d3f217ea"),
        "text" : "Me, my dad, and my brother always get the exact order of food.",
        "user" : {
            "screen_name" : "CowlonFullerton",
            "geo_enabled" : true,
            "statuses_count" : 74382,
            "followers_count" : 1808,
        },
        "coordinates" : {
            "type" : "Point",
            "coordinates" : [
                -96.87688668,
                37.81792338
            ]
        },
        "place" : {
            "place_type" : "city",
            "name" : "El Dorado",
            "full_name" : "El Dorado, KS",
            "country" : "United States",
        },
        "entities" : {
            "hashtags" : [ ],
        },
    }
        
Indexes
-------

The following compound index is in place for testing purely based on 
geocoded information within each post.  Depending on the amount of 
data it may be a good idea to extend this index to another field that 
will be used heavily by the application.  For now we will keep it 
simple and use cursor.explain_ later on to see how much scanning is 
being done to each index.

..  code :: javascript    

    db.tweets.ensureIndex({
        "place.country": 1,
        "place.full_name": 1
    });
    
The Problem
===========

Based on a user preference we want to query all users that have more 
than 500 followers that have made a post recently from one major city 
to the next and then eventually the entire country.  We will just 
assume that documents in the collection are 'recent', perhaps by using 
a TTL_ special index.

The user has the following preference:

* The city ``Los Angeles, CA``
* The city ``Manhattan, NY``
* The city ``Philadelphia, PA``
* The city ``Chicago, IL``
* The city ``Houston, TX``
* The country ``United States``

The Solution
============

Building a query for that using or_ is relatively easy since we know 
exactly what we want to search for.  From the API standpoint the 
language needs to append dictionary or SON_ objects to the ``$or`` 
field in order.  For the following example query we will turn on 
cursor.explain_ with ``verbose`` toggled on.

..  code-block :: javascript

    db.tweets.find({
        '$or': [{
            'place.country': 'United States',
            'place.full_name': 'Los Angeles, CA',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Manhattan, NY',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Philadelphia, PA',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Chicago, IL',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Houston, TX',
        }, {
            'place.country': 'United States',
        }]
    }).explain(verbose = true)


Since we used or_ we have a ``clauses`` array that specifies the query 
plans being used.  Each clause should look familiar to users that are 
experiences with the output of cursor.explain_.

..  code-block :: javascript

    // Simplified
    
    {
        "clauses" : [
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 265,
                "nscannedObjects" : 265,
                "nscanned" : 265,
                "millis" : 2,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Los Angeles, CA",
                            "Los Angeles, CA"
                        ]
                    ]
                },
            },
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 246,
                "nscannedObjects" : 246,
                "nscanned" : 246,
                "millis" : 11,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Manhattan, NY",
                            "Manhattan, NY"
                        ]
                    ]
                },
            },
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 202,
                "nscannedObjects" : 202,
                "nscanned" : 202,
                "millis" : 10,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Philadelphia, PA",
                            "Philadelphia, PA"
                        ]
                    ]
                },
            },
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 168,
                "nscannedObjects" : 168,
                "nscanned" : 168,
                "millis" : 5,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Chicago, IL",
                            "Chicago, IL"
                        ]
                    ]
                },
            },
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 148,
                "nscannedObjects" : 148,
                "nscanned" : 148,
                "millis" : 6,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Houston, TX",
                            "Houston, TX"
                        ]
                    ]
                },
            },
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 17906,
                "nscannedObjects" : 18935,
                "nscanned" : 18935,
                "millis" : 884,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            {
                                "$minElement" : 1
                            },
                            {
                                "$maxElement" : 1
                            }
                        ]
                    ]
                },
            }
        ],
        "n" : 18935,
        "nscannedObjects" : 19964,
        "nscanned" : 19964,
        "millis" : 920,
        "server" : "buckaroobanzai:27017"
    }
    
That's a lot of documents!, thankfully we can request that the user do 
some pagination or fetch the cursor in batches.  The above information 
shows that ``Los Angeles, CA`` has 265 documents associated with it 
and ``Manhattan, NY`` has 246.  If the user set their document limit 
to **500** they would only hit the first two clauses and of course the 
query would be nice and fast.

..  code :: javascript

    {
        "clauses" : [
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 265,
                "nscannedObjects" : 265,
                "nscanned" : 265,
                "millis" : 2,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Los Angeles, CA",
                            "Los Angeles, CA"
                        ]
                    ]
                },
            },
            {
                "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                "n" : 235,
                "nscannedObjects" : 235,
                "nscanned" : 235,
                "millis" : 10,
                "indexBounds" : {
                    "place.country" : [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name" : [
                        [
                            "Manhattan, NY",
                            "Manhattan, NY"
                        ]
                    ]
                },
            }
        ],
        "n" : 500,
        "nscannedObjects" : 500,
        "nscanned" : 500,
        "millis" : 12,
        "server" : "buckaroobanzai:27017"
    }

This is right in line with how hierarchical storage management_ is 
done.  If we are clever we can isolate low traffic index ranges to 
less expensive shard servers and use this solution to only hit those 
servers if the rest of the shards could not completely satisfy the 
query.

As previously stated, the user wants to include only documents posted 
by individuals that have more than 500 followers.  We can do this one 
of two ways depending on how flexible we want this query.

..  code-block :: javascript

    db.tweets.find({
        '$or': [{
            'place.country': 'United States',
            'place.full_name': 'Los Angeles, CA',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Manhattan, NY',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Philadelphia, PA',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Chicago, IL',
        }, {
            'place.country': 'United States',
            'place.full_name': 'Houston, TX',
        }, {
            'place.country': 'United States',
        }],
        'user.followers_count': { '$gte': 500 },
    }).limit(500).explain(verbose = true)

..  code-block :: javascript

    db.tweets.find({
        '$or': [{
            'place.country': 'United States',
            'place.full_name': 'Los Angeles, CA',
            'user.followers_count': { '$gte': 500 },
        }, {
            'place.country': 'United States',
            'place.full_name': 'Manhattan, NY',
            'user.followers_count': { '$gte': 500 },
        }, {
            'place.country': 'United States',
            'place.full_name': 'Philadelphia, PA',
            'user.followers_count': { '$gte': 500 },
        }, {
            'place.country': 'United States',
            'place.full_name': 'Chicago, IL',
            'user.followers_count': { '$gte': 500 },
        }, {
            'place.country': 'United States',
            'place.full_name': 'Houston, TX',
            'user.followers_count': { '$gte': 500 },
        }, {
            'place.country': 'United States',
            'user.followers_count': { '$gte': 500 },
        }],
    }).limit(500).explain(verbose = true)

The latter query allows us to change ``user.followers_count`` to match 
any limit the user requests.  Perhaps they want to scan the country 
for any individuals with over 10000 followers.

Pagination
----------

Without going to far into it.  If you're client side can tell you 
where it last left off (say.. the middle of ``Manhattan, NY``) your 
client side code can simply leave ``Los Angeles, CA`` out of the loop.  
Unfortunately since your clauses aren't individually sorted (see 
Gotchas_) it can be a bit difficult to pick up where you left off 
without also knowing how many documents into ``Manhattan, NY`` the 
last query got to.
       
Geospatial Queries
------------------

I've written a post called Geospatial MongoDB using Quadtrees and 
Geohashes_

Gotchas
-------

There are of course a few gotchas with using this solution.

No more sort_!
~~~~~~~~~~~~~~

Go ahead and try it.  Instead of processing each or_ clause and 
returning sorted chunks you will instead process the index directly 
(hopefully) and filter the results through the or_ array using a 
post-processor.

You have to do more client side code!
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I couldn't be happier about that.  Making specific use of a very 
simple database solution (comparitively speaking) is going to 
eventually require some pre and post processing by the client if you 
want to do anything that isn't directly supported.  Thankfully 
MongoDB_ is very **streamy** and processing a cursor in most languages 
is very simple.

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

..  _hierarchial storage management: http://en.wikipedia.org/wiki/Hierarchical_Storage_Management

..  _Geospatial MongoDB using Quadtrees and Geohashes: geospatial-mongodb-using-quadtrees-and-geohashes.rst
