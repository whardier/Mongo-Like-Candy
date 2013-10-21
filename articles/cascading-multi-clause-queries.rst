==============================
Cascading Multi-Clause Queries
==============================

:Author: Shane R. Spencer <shane@bogomip.com>
:Date: Sun Oct 20 01:14:35 UTC 2013

.. contents::

..  _$or: http://docs.mongodb.org/manual/reference/operator/or/

..  _cursor.limit(): http://docs.mongodb.org/manual/reference/method/cursor.limit/

..  _cursor.sort(): http://docs.mongodb.org/manual/reference/method/cursor.sort/

..  _cursor.skip(): http://docs.mongodb.org/manual/reference/method/cursor.skip/

..  _cursor.explain(): http://docs.mongodb.org/manual/reference/method/cursor.explain/

..  _cursor.explain().clauses: http://docs.mongodb.org/manual/reference/method/cursor.explain/#or-query-output-fields

..  _mongodb: http://www.mongodb.org/

..  _2d: http://docs.mongodb.org/manual/core/2d/

..  _2dsphere: http://docs.mongodb.org/manual/core/2dsphere/

..  _mongoimport: http://docs.mongodb.org/manual/reference/program/mongoimport/

..  _geojson: http://docs.mongodb.org/manual/reference/glossary/#term-geojson

..  _json: http://docs.mongodb.org/manual/reference/glossary/#term-json

..  _hierarchical storage management: http://en.wikipedia.org/wiki/Hierarchical_storage_management

..  _twitter: http://twitter.com/

..  _twitter streaming api: https://dev.twitter.com/docs/streaming-apis

..  _compound indexes: http://docs.mongodb.org/manual/core/index-compound

Preface
=======

`MongoDB`_ definitely encourages developers to think outside of the relational database box and create some clever query optimization's that allow `MongoDB`_ to operate at its peak performance.  This includes finding ways to reduce table scans, discovering the right index for the job, and using some lesser known optimization's like the `$or`_ logical query operator to do what I have been calling ``Cascading Multi-Clause Queries``.

Multi-clause queries are done by way of the `$or`_ logical query operator.  When using `$or`_ on a query `cursor.explain()`_ returns a bit of extra information in the form of `cursor.explain().clauses`_ which is an ordered list of the sort of output you would expect from `cursor.explain()`_ for each clause of the query.

``Cascading`` refers to organizing each step of a multi-clause query so that certain index regions and table scans are processed in a specific order.  Usage of the `cursor.limit()`_ method allows for the query to exit without processing all the clauses if the limit size is reached. Ideally each clause would have an index associated with it.

Here is a 2 clause query from the official `MongoDB documentation <http://docs.mongodb.org/manual/reference/operator/query/or/#op._S_or>`_ where ``price`` is part of the first query clause and ``sale`` is part of the second while ``qty`` further filters the results of each:

..  code:: javascript

    db.inventory.find({
        '$or': [{
            price: 1.99
        }, {
            sale: true
        }],
        qty: {
            '$in': [20,
                50
            ]
        }
    });

If there are 100 inventory items with a price of ``1.99`` that match the ``qty`` filter and we limit the overall query to 100 documents then the ``price`` clause will completely satisfy the query and any further query processing will be ignored.  This is a good example of how ``cascading`` is being put to use by returning documents in clause processed order which turns out to be an amazing optimization for developers interested in implementing priority based queries or `hierarchical storage management`_ may be required.

Optimization's that benefit from `$or`_:

* An index pyramid where more focused sparse and small indexes are queried before others.

* Priority cascading user defined or hash based geo queries without utilizing `2d`_ or `2dsphere`_ indexes.

* Pseudo-sorting very large volumes of data without requiring using `cursor.sort()`_.

* Removing clauses from pagination results as the result set grows to support faster `cursor.skip()`_ operations.
    
Summary
=======

I inadvertently found out about multi-clause queries while examining a well crafted query that I was feeling incredibly hopeful about.  For this query I was looking to create a single stream of documents by querying different values of a ``location`` field and I ended up using `$or`_ to see how it would react.  The goal for me was to first query documents with a specific location and then all the surrounding cities in an order I had determined before making the query.

To take advantage of this I knew I would need to feed the query an ordered set of locations which I would pregenerate based on my own algorithms based on the application users preferences.

Logically, `$or`_ performed ordered queries where I was wrongly thinking of it as a post-filter for an index scan.

Data
====

This article focuses on using a sample stream of geographically referenced Twitter posts using the `Twitter Streaming API`_.  It may not surprise any of you that the `JSON`_ output from `Twitter`_ can be directly imported into `MongoDB`_ using `mongoimport`_ and contains valid `GeoJSON`_ for direct use with the `2dsphere`_ geospatial index as well as an array of points that works well with the `2d`_ geospatial index.

`Twitter`_ posts make an **excellent** data source to use when testing indexing requirements like multi-lingual text searching, geospatial data, `compound indexes`_, and works very well when you simply need a lot of very unique data to play with.

Sample `Twitter`_ Post
----------------------

Geographically referenced `Twitter`_ posts contain location information through the ``place`` field which focuses on the nearest city and state information for the ``coordinates`` field that defines where on earth the post was approximately made from.

..  code:: javascript

    db.tweets.findOne({
        'place.full_name': 'Los Angeles, CA'
    }, {
        'text': true,
        'user.screen_name': true,
        'coordinates': true,
        'place.full_name': true,
        'place.country': true
    });
    
    {
        "_id": ObjectId("52647c32b7c03befed384f00"),
        "text": "Time is going by so fast.",
        "user": {
            "screen_name": "DoctorWhomz"
        },
        "coordinates": {
            "type": "Point",
            "coordinates": [-118.18497793,
                34.08546991
            ]
        },
        "place": {
            "full_name": "Los Angeles, CA",
            "country": "United States"
        }
    }
        
Indexes
-------

The following `compound index`_ is in place for testing purely based on the geographical information within each post.  Depending on the amount of data it may be a good idea to extend this index to another field that will be used heavily by the application.  For now we will keep it simple and use `cursor.explain()`_ later on to see how much scanning is being done to each index.

..  code:: javascript    

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
a TTL special index.

The user has the following preference:

* The city ``Los Angeles, CA``
* The city ``Manhattan, NY``
* The city ``Philadelphia, PA``
* The city ``Chicago, IL``
* The city ``Houston, TX``
* The country ``United States``

The Solution
============

Building a query for that using or is relatively easy since we know 
exactly what we want to search for.  From the API standpoint the 
language needs to append dictionary or SON objects to the `$or`_
field in order.  For the following example query we will turn on 
cursor.explain with ``verbose`` toggled on.

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


Since we used or we have a ``clauses`` array that specifies the query 
plans being used.  Each clause should look familiar to users that are 
experiences with the output of cursor.explain.

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

..  code:: javascript

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

This is right in line with how `hierarchical storage management`_ is 
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

Keep in mind that when want your or operator to be first in a query 
you should always use SON objects to build your query.  This makes 
sure that the query document is ordered properly when using a 
programming language where dictionaries have no ordering, like 
Python.

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

In my article `Geospatial MongoDB using Quadtrees and Geohashes 
<geospatial-mongodb-using-quadtrees-and-geohashes.rst>` I go over 
using hashes that narrow down on specific locations the longer the 
hash string becomes which is known as the precision.  Pulling off a 
query where I look for all points within a specific location is pretty 
simple and using the or operator makes it simple to get a roughly 
distance sorted result set without using 2d or 2dsphere geospatial 
indexes.

Why?  Because 2d_/2dsphere indexes cannot be used as shard keys 
however geohash and quadtree strings can.

Lets pull off the following:

* query a hash the size of a house
* query the hashes neighbors
* query a hash the size of a block
* query the hashes neighbors

..  code:: javascript

    db.tweets.find({
        '$or': [{
            'geohash': /^bdvkjqwr/,
        }, {
            'geohash': {
                '$in': [
                    /^bdvkjqy0/,
                    /^bdvkjqy2/,
                    /^bdvkjqy8/,
                    /^bdvkjqwp/,
                    /^bdvkjqwx/,
                    /^bdvkjqwn/,
                    /^bdvkjqwq/,
                    /^bdvkjqww/,
                ]
            }
        }, {
            'geohash': /^bdvkjq/,
        }, {
            'geohash': {
                '$in': [
                    /^bdvkjp/,
                    /^bdvkjr/,
                    /^bdvkjx/,
                    /^bdvkjn/,
                    /^bdvkjw/,
                    /^bdvkjj/,
                    /^bdvkjm/,
                    /^bdvkjt/,
                ]
            }
        }],
    }).limit(500).explain(verbose = true)

Gotchas
=======

There are of course a few gotchas with using this solution.

No more cursor.sort
-------------------

Go ahead and try it.  Instead of processing each or clause and 
returning sorted chunks you will instead process the index directly 
(hopefully) and filter the results through the or array using a 
post-processor.

You have to do more client side code
------------------------------------

I couldn't be happier about that.  Making specific use of a very 
simple database solution (comparatively speaking) is going to 
eventually require some pre and post processing by the client if you 
want to do anything that isn't directly supported.  Thankfully 
MongoDB is very **streamy** and processing a cursor in most languages 
is very simple.

Extra Info
==========

Also check out `Interim Tables F.T.W. <interim-tables-ftw.rst>` to see 
how the result set for or based cascading multi-clause queries can be 
stored into an interim table and a secondary query can be done against 
the data.  Both solutions are a killer combination when it comes to 
keeping index size down and creating simple and straight forward data 
sets highly searchable and easily paginated.

