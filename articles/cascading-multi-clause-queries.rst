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

..  _cursor.find(): http://docs.mongodb.org/manual/reference/method/cursor.find/

..  _cursor.hint(): http://docs.mongodb.org/manual/reference/method/cursor.hint/

..  _cursor.skip(): http://docs.mongodb.org/manual/reference/method/cursor.skip/

..  _cursor.explain(): http://docs.mongodb.org/manual/reference/method/cursor.explain/

..  _cursor.explain().clauses: http://docs.mongodb.org/manual/reference/method/cursor.explain/#or-query-output-fields

..  _db.collection.find(): http://docs.mongodb.org/manual/reference/method/db.collection.find/

..  _mongodb: http://www.mongodb.org/

..  _2d: http://docs.mongodb.org/manual/core/2d/

..  _2dsphere: http://docs.mongodb.org/manual/core/2dsphere/

..  _mongoimport: http://docs.mongodb.org/manual/reference/program/mongoimport/

..  _geojson: http://docs.mongodb.org/manual/reference/glossary/#term-geojson

..  _json: http://docs.mongodb.org/manual/reference/glossary/#term-json

..  _sparse indexes: http://docs.mongodb.org/manual/core/index-sparse/

..  _sparse index: http://docs.mongodb.org/manual/core/index-sparse/

..  _twitter: http://twitter.com/

..  _twitter streaming api: https://dev.twitter.com/docs/streaming-apis

..  _compound indexes: http://docs.mongodb.org/manual/core/index-compound

..  _compound index: http://docs.mongodb.org/manual/core/index-compound

..  _index order: http://docs.mongodb.org/manual/tutorial/sort-results-with-indexes/

..  _geohashes: http://en.wikipedia.org/wiki/Geohash

..  _quadtrees: http://en.wikipedia.org/wiki/Quadtree

Preface
=======

In `MongoDB`_ multi-clause queries are done by way of the `$or`_ logical query operator.

**Cascading** refers to the application level preference of a multi-clause query so that certain index regions and table scans are processed in a specific order.  Furthermore, the application can choose to use the `cursor.limit()`_ method that allows multi-clause queries to exit early without processing all the clauses if the limit is reached.  What a wonderful optimization for large data sets.

Here is a 2 clause query from the official `MongoDB documentation <http://docs.mongodb.org/manual/reference/operator/query/or/#op._S_or>`_ where **price** is part of the first query clause and **sale** is part of the second while **qty** further filters the results of each:

..  code-block :: javascript

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

Simply put... if there are **100** inventory items with a price of **1.99** that match the **qty** filter and we limit the overall query to **100** documents then the **price** clause will completely satisfy the query and any further query processing will be ignored.  This is a good example of how **cascading** is being put to use by returning documents in clause processed order.

Summary
=======

I inadvertently found out about multi-clause queries while examining a well crafted query that I was feeling incredibly hopeful about.  For this query I was looking to create a single stream of documents by querying different values of a **location** field and I ended up using `$or`_ to see how it would react.  The goal for me was to first query documents with a specific location and then all the surrounding cities in an order I had determined before making the query.

To take advantage of this I knew I would need to feed the query an ordered set of locations which I would pregenerate based on my own algorithms based on the application users preferences.

Logically, `$or`_ performed multiple queries in order where I was wrongly thinking of it as a post-filter for a single query. Amazed, it finally clicked that there was an opportunity be able extend a single query into a concatenation of multiple queries with a single execution of `db.collection.find()`_.

Data
====

This article focuses on using a sample stream of geographically referenced Twitter posts using the `Twitter Streaming API`_ which is a streaming JSON feed that easily imports into `MongoDB`_.

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

The following `compound index`_ is in place for test queries that will be looking at the geographic information within each post.

..  code-block :: javascript    

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

The Solution
============

Building a query to deal with explicitly defined ordering using `$or`_ is relatively easy since we know exactly what we want to search for.  From the API standpoint the language needs to append dictionary or SON objects to the `$or`_ field in order.  For the following example query we will turn on cursor.explain with **verbose** toggled on.

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

As previously stated, the user wants to include only documents posted by individuals that have more than **500** followers.  We can do this one of two ways depending on how flexible we want this query.

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

Each clause can rely on a different indexes.  However, there's no method of applying a `cursor.hint()`_ to individual clauses.

For instance if you wanted to use a `sparse index`_ in the first clause but wanted to use a `compound index`_ for the rest of them then you would want to specifically query around whatever fields are involved with the index you want to use.

..  code-block :: javascript    

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

Sorting Clause Results
----------------------

Using the `cursor.sort()`_ method on multi-clause query will inevitably fail if there are too many documents returned by the query.  For efficiency sake queries that use an index and sort on that indexes fields will be returned in `index order`_ eliminating post-sorting of a buffer of information.

`cursor.sort()`_ just isn't optimized for multi-clause queries full of multiple query plans and I don't believe it should be.  Using multiple clauses allows us to fetch different parts of the same document set in varying orders according to what we need.  Sorting the results after the documents are returned may go against your reasons for using `$or`_ in the first place.

For the most part if each clause's query plan is using an index the documents retrieved for that clause will returned through an index scan in `index order`_.  If no field specific index is being used for a clause then document ordering will most likely be difficult to predict.

Each clause's query plan attempts to find the most relevant index for the given query parameters and conditions. A clause is in essense it's own individualized `cursor.find()`_ command and gets the benefit of index `index order`_ when returning documents.

Therefore, using the `index order`_ of an index seems to be the only obvious way to make each query sorted. This is exactly where a well tuned `compound index`_ can help keep these tweets in stay in an order we would prefer for a specific clause.

..  code-block :: javascript    

    // place.country_1_place.full_name_1_user.screen_name_1
    db.tweets.ensureIndex({
        "place.country": 1,
        "place.full_name": 1,
        "user.screen_name": 1
    });

In order to make sure a clause uses this index the conditions simply need to require that **user.screen_name** be `$gte`_ the lowest possible string value.  Doing so will is enough of a hint to the query planner to create a query plan that will target this index.  Any query plan that chooses this specific index will return documents in index ascending order starting with **place.country**, followed by **place.full_name**, and finally **user.screen_name**.

Conclusion
==========

`MongoDB`_ definitely encourages developers to think outside of the relational database box and create some clever query optimizations that allow `MongoDB`_ to operate at its peak performance.  This includes finding ways to reduce table scans, discovering the right index for the job, and using some lesser known optimizations like the `$or`_ logical query operator to do what I have been calling **Cascading Multi-Clause Queries**.

