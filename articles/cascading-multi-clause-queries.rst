==============================
Cascading Multi-Clause Queries
==============================

:Author: Shane R. Spencer <shane@bogomip.com>
:Date: Sun Oct 20 01:14:35 UTC 2013

.. contents::

..  _$or: http://docs.mongodb.org/manual/reference/operator/or/

..  _$gte: http://docs.mongodb.org/manual/reference/operator/query/gte/

..  _cursor.limit(): http://docs.mongodb.org/manual/reference/method/cursor.limit/

..  _cursor.sort(): http://docs.mongodb.org/manual/reference/method/cursor.sort/

..  _cursor.hint(): http://docs.mongodb.org/manual/reference/method/cursor.hint/

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

..  _sparse indexes: http://docs.mongodb.org/manual/core/index-sparse/

..  _sparse index: http://docs.mongodb.org/manual/core/index-sparse/

..  _twitter: http://twitter.com/

..  _twitter streaming api: https://dev.twitter.com/docs/streaming-apis

..  _compound indexes: http://docs.mongodb.org/manual/core/index-compound

..  _compound index: http://docs.mongodb.org/manual/core/index-compound

..  _natural order: http://docs.mongodb.org/manual/reference/glossary/#term-natural-order

..  _tag aware sharding: http://docs.mongodb.org/manual/core/tag-aware-sharding/

..  _shard key: http://docs.mongodb.org/manual/core/sharding-shard-key/

..  _geohash: http://en.wikipedia.org/wiki/Geohash

..  _geohashes: http://en.wikipedia.org/wiki/Geohash

Preface
=======

`MongoDB`_ definitely encourages developers to think outside of the relational database box and create some clever query optimization's that allow `MongoDB`_ to operate at its peak performance.  This includes finding ways to reduce table scans, discovering the right index for the job, and using some lesser known optimization's like the `$or`_ logical query operator to do what I have been calling **Cascading Multi-Clause Queries**.

Multi-clause queries are done by way of the `$or`_ logical query operator.  When using `$or`_ on a query `cursor.explain()`_ returns a bit of extra information in the form of `cursor.explain().clauses`_ which is an ordered list of query plans per clause that will be executed.  For more information about query plans please see `cursor.explain()`_'s documentation in the official `MongoDB`_ documentation where they are summarized as "The query plan is the plan the server uses to find the matches for a query.".

**Cascading** refers to the application level preference of a multi-clause query so that certain index regions and table scans are processed in a specific order.  Furthermore, the application can choose to use the `cursor.limit()`_ method that allows multi-clause queries to exit early without processing all the clauses if the limit is reached.  What a wonderful optimization for large data sets.

Here is a 2 clause query from the official `MongoDB documentation <http://docs.mongodb.org/manual/reference/operator/query/or/#op._S_or>`_ where **price** is part of the first query clause and **sale** is part of the second while **qty** further filters the results of each:

..  code:: javascript

    db.inventory.find({
        "$or": [{
            price: 1.99
        }, {
            sale: true
        }],
        qty: {
            "$in": [20,
                50
            ]
        }
    });

If there are 100 inventory items with a price of **1.99** that match the **qty** filter and we limit the overall query to 100 documents then the **price** clause will completely satisfy the query and any further query processing will be ignored.  This is a good example of how **cascading** is being put to use by returning documents in clause processed order which turns out to be an amazing optimization for developers interested in implementing priority based queries or `hierarchical storage management`_ may be required.

Optimization's that benefit from `$or`_:

* An series of index scans where more focused `sparse indexes`_ or `compound indexes`_ are queried before others.

* Index Pyramids for hash based value queries and geospatial queries without utilizing `2d`_ or `2dsphere`_ indexes.

* Pseudo-sorting very large volumes of data without requiring using `cursor.sort()`_.

* Removing clauses from pagination results as the result set grows to support faster `cursor.skip()`_ operations.
    
Summary
=======

I inadvertently found out about multi-clause queries while examining a well crafted query that I was feeling incredibly hopeful about.  For this query I was looking to create a single stream of documents by querying different values of a **location** field and I ended up using `$or`_ to see how it would react.  The goal for me was to first query documents with a specific location and then all the surrounding cities in an order I had determined before making the query.

To take advantage of this I knew I would need to feed the query an ordered set of locations which I would pregenerate based on my own algorithms based on the application users preferences.

Logically, `$or`_ performed ordered queries where I was wrongly thinking of it as a post-filter for an index scan.

Data
====

This article focuses on using a sample stream of geographically referenced Twitter posts using the `Twitter Streaming API`_.  It may not surprise any of you that the `JSON`_ output from `Twitter`_ can be directly imported into `MongoDB`_ using `mongoimport`_ and contains valid `GeoJSON`_ for direct use with the `2dsphere`_ geospatial index as well as an array of points that works well with the `2d`_ geospatial index.

`Twitter`_ posts make an **excellent** data source to use when testing indexing requirements like multi-lingual text searching, geospatial data, `compound indexes`_, and works very well when you simply need a lot of very unique data to play with.

Sample `Twitter`_ Post
----------------------

Geographically referenced `Twitter`_ posts contain location information through the **place** field which focuses on the nearest city and state information for the **coordinates** field that defines where on earth the post was approximately made from.

..  code:: javascript

    db.tweets.findOne({
        "place.full_name": "Los Angeles, CA"
    }, {
        "text": true,
        "user.screen_name": true,
        "coordinates": true,
        "place.full_name": true,
        "place.country": true
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

    // place.country_1_place.full_name_1
    db.tweets.ensureIndex({
        "place.country": 1,
        "place.full_name": 1
    });
    
The Problem
===========

Based on the applications users preference we want to query all twitter users that have more than 500 followers and have made a post recently from one major city to the next and then eventually the entire country.

The user has the following preference:

* **Los Angeles, CA**

* **Manhattan, NY**

* **Philadelphia, PA**

* **Chicago, IL**

* **Houston, TX**

* and finally simply **United States**

We want the results to return in this order, but not specifically ordered otherwise we would need to create a sort key that matched the users preference.  Eventually we want to be able to use the default `natural order`_ of documents in each clauses related indexes so that the documents relating to **Manhattan, NY** come after **Los Angeles, CA** but are still sorted by another key.

The Solution
============

Building a query for that using or is relatively easy since we know exactly what we want to search for.  From the API standpoint the language needs to append dictionary or SON objects to the `$or`_ field in order.  For the following example query we will turn on cursor.explain with **verbose** toggled on.

Since we used `$or`_ we will have a **clauses** array that specifies the clauses and the query plans being used.

..  code-block :: javascript
    
    db.tweets.find({   
        "$or": [{       
            "place.country": "United States",
            "place.full_name": "Los Angeles, CA",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Manhattan, NY",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Philadelphia, PA",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Chicago, IL",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Houston, TX",
               
        }, {       
            "place.country": "United States"   
        }]
    }).explain(verbose = true);

    // Shortened and Simplified
    {
        "clauses": [{
            "allPlans": [{
                "cursor": "BtreeCursor place.country_1_place.full_name_1",
                "n": 38,
                "nscannedObjects": 38,
                "nscanned": 38,
                "indexBounds": {
                    "place.country": [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name": [
                        [
                            "Los Angeles, CA",
                            "Los Angeles, CA"
                        ]
                    ]
                }
            }]
        }, {
            "allPlans": [{
                "cursor": "BtreeCursor place.country_1_place.full_name_1",
                "n": 25,
                "nscannedObjects": 25,
                "nscanned": 25,
                "indexBounds": {
                    "place.country": [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name": [
                        [
                            "Manhattan, NY",
                            "Manhattan, NY"
                        ]
                    ]
                }
            }]
        }, {
            /* ... */
        }, {
            "allPlans": [{
                "cursor": "BtreeCursor place.country_1_place.full_name_1",
                "n": 2070,
                "nscannedObjects": 2188,
                "nscanned": 2188,
                "indexBounds": {
                    "place.country": [
                        [
                            "United States",
                            "United States"
                        ]
                    ],
                    "place.full_name": [
                        [{
                            "$minElement": 1
                        }, {
                            "$maxElement": 1
                        }]
                    ]
                }
            }]
        }],
        "n": 2188,
        "nscannedObjects": 2306,
        "nscanned": 2306,
        "nscannedObjectsAllPlans": 2306,
        "nscannedAllPlans": 2306,
        "millis": 76,
        "server": "buckaroobanzai:27017"
    }
            
That's a lot of documents and since we are working with potentially live `Twitter`_ data we know it's going to grow like crazy.  Thankfully we can request that the user do some pagination if they want to see all the documents.  The above information shows that **Los Angeles, CA** has **38** tweet documents associated with it and **Manhattan, NY** has **25**.  If the application limits each page to **50** documents per page the cursor would only fetch documents from the first two clauses for the first page.

..  code:: javascript

    db.tweets.find({   
        "$or": [{       
            "place.country": "United States",
            "place.full_name": "Los Angeles, CA",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Manhattan, NY",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Philadelphia, PA",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Chicago, IL",
               
        }, {       
            "place.country": "United States",
            "place.full_name": "Houston, TX",
               
        }, {       
            "place.country": "United States"   
        }]
    }).limit(50).explain(verbose = true);
    
    // Shortened and Simplified
    {
        "clauses" : [
            {
                "allPlans" : [
                    {
                        "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                        "n" : 38,
                        "nscannedObjects" : 38,
                        "nscanned" : 38,
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
                        }
                    }
                ]
            },
            {
                "allPlans" : [
                    {
                        "cursor" : "BtreeCursor place.country_1_place.full_name_1",
                        "n" : 12,
                        "nscannedObjects" : 12,
                        "nscanned" : 12,
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
                        }
                    }
                ]
            }
        ],
        "n" : 50,
        "nscannedObjects" : 50,
        "nscanned" : 50,
        "nscannedObjectsAllPlans" : 50,
        "nscannedAllPlans" : 50,
        "millis" : 0,
        "server" : "buckaroobanzai:27017"
    }

I have a lot of appreciation for **millis: 0**.

This is right in line with how `hierarchical storage management`_ is done.  If this collection were sharded, which it probably should be, we have the opportunity to be clever and isolate low traffic index ranges to less expensive shard servers and use this solution to only hit those servers if the rest of the shards could not completely satisfy the query.  The gotcha is in the `shard key`_ and making sure that each clause defines it explicitely by making sure those fields are part of the query.  Doing so provides an alternative to `tag aware sharding`_ as well as a welcome compliment to it.

As previously stated, the user wants to include only documents posted by individuals that have more than **50** followers.  We can do this one of two ways depending on how flexible we want this query.

..  code-block :: javascript

    db.tweets.find({
        "$or": [{
            "place.country": "United States",
            "place.full_name": "Los Angeles, CA",
        }, {
            "place.country": "United States",
            "place.full_name": "Manhattan, NY",
        }, {
            "place.country": "United States",
            "place.full_name": "Philadelphia, PA",
        }, {
            "place.country": "United States",
            "place.full_name": "Chicago, IL",
        }, {
            "place.country": "United States",
            "place.full_name": "Houston, TX",
        }, {
            "place.country": "United States",
        }],
        "user.followers_count": { "$gte": 500 },
    }).limit(50).explain(verbose = true)

..  code-block :: javascript

    db.tweets.find({
        "$or": [{
            "place.country": "United States",
            "place.full_name": "Los Angeles, CA",
            "user.followers_count": { "$gte": 500 },
        }, {
            "place.country": "United States",
            "place.full_name": "Manhattan, NY",
            "user.followers_count": { "$gte": 500 },
        }, {
            "place.country": "United States",
            "place.full_name": "Philadelphia, PA",
            "user.followers_count": { "$gte": 500 },
        }, {
            "place.country": "United States",
            "place.full_name": "Chicago, IL",
            "user.followers_count": { "$gte": 500 },
        }, {
            "place.country": "United States",
            "place.full_name": "Houston, TX",
            "user.followers_count": { "$gte": 500 },
        }, {
            "place.country": "United States",
            "user.followers_count": { "$gte": 500 },
        }],
    }).limit(50).explain(verbose = true)

The latter query allows us to change **user.followers_count** to match any limit the user requests for each region.  Perhaps they want to scan the country for any individuals with over 10000 followers.

Multi-Index Support
-------------------

Each clause can rely on a different indexes or even force a table scan.  There's no method of applying a `cursor.hint()`_ to individual clauses so the magic is all in what fields you want to search on.  Make the server make an optimal assumption as to what index to use.

For instance if you wanted to use a `sparse index`_ in the first clause but wanted to use a `compound index`_ for the rest of them then you would want to specifically query around whatever fields are involved with the index you want to use.

..  code:: javascript    

    // user.screen_name_1
    db.tweets.ensureIndex({
        "user.screen_name": 1,
    }, {
        "sparse": true
    });

    db.tweets.find({   
        "$or": [{
            "user.screen_name": "DoctorWhomz",
        }, {       
            "place.country": "United States",
            "place.full_name": "Houston, TX",
               
        }, {       
            "place.country": "United States"   
        }]
    }).explain(verbose = true);

In the example above the following indexes will be used in order:

* **user.screen_name_1** (sparse)

* **place.country_1_place.full_name_1**

* **place.country_1_place.full_name_1**

Sorting
-------

Using the `natural order`_ of an index seems to be the only obvious way to make each query sorted, therefore a very useful default `compound index`_ can help keep these tweets in order.  Literally.

..  code:: javascript    

    // place.country_1_place.full_name_1_user.screen_name_1
    db.tweets.ensureIndex({
        "place.country": 1,
        "place.full_name": 1,
        "user.screen_name": 1
    });

Remove **place.country_1_place.full_name_1** or keep it and simply require that **user.screen_name** be `$gte`_ the lowest possible string value and the query plans will target this index for use. Any query plan that chooses this index will return documents in index ascending order starting with **place.country**, followed by **place.full_name**, and finally **user.screen_name**.

Pagination `cursor.skip()`_ Optimization
----------------------------------------

This method offers a somewhat unique opportunity to leave out the clause for **Los Angeles, CA** if the application notices there are no more **Los Angeles, CA** oriented documents in the result set.  With a little counting on the application side the `cursor.skip()`_ method can be reduced by how many documents existed in a clause that is going to be removed and the overall query benefits by not having to skip through clauses that are no longer valid.

Index Pyramids
--------------

Index pyramids refer to the ability to query for more specific data on a specific field and then further expand the boundaries of the query.  This technique tuned toward using a specific field to help quickly get at relevant information and then eventually scan a larger index range to finish if more results are requested.

For example lets look for all `Twitter`_ posts that are created by the **user.screen_name** "whardier" followed values starting with "whard" and eventually just "w":

..  code:: javascript

    db.tweets.find({
        "$or": [{
            "user.screen_name": { "whardier" },
        }, {
            "user.screen_name": { /^whard/ },
        }, {
            "user.screen_name": { /^w/ },
        }],
    })

The **user.screen_name_1** index will be used 3 different times in this query.

As for geospatial pyramids, `Geohashes`_ are pyramids defining geospatial areas. The longer the hash the narrower the area relative to the first parts of the hash.

Currently `MongoDB`_ sharding does not allow `2d`_ or `2dsphere`_ hashes to be part of a `shard key`_ and geospatially aware hashes like `Geohashes`_ can help compensate for this, as well as offer multi-clause area based queries.

Lets pull off the following:

* Query a hash the size of a house

* Query the hashes direct neighbors

* Query a grandparent hash

..  code:: javascript

    db.tweets.find({
        "$or": [{
            "geohash": /^bdvkjqwr/,
        }, {
            "geohash": {
                "$in": [
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
            "geohash": /^bdvkj/ 
        }],
    })

