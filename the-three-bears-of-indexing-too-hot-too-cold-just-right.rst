:Author: Shane R. Spencer <shane@bogomip.com>

==========================================================
The Three Bears of Indexing: Too Hot, Too Cold, Just Right
==========================================================

.. contents :: The Story
    :backlinks: entry

Preface
=======

Many of us know the story of `Goldilocks and the Three Bears 
<http://en.wikipedia.org/wiki/The_Story_of_the_Three_Bears>`_.  I often 
think of the story when I'm sniffing out the right indexing that needs to 
be done for new projects.

Sometimes the indexing is too hot meaning that the index being tested is 
too large to be easily managed and is taking up more memory than 
necessary, or too cold where the index is too small forcing larger than 
desirable scans of the documents.  The mix between the two is sometimes 
just right when I find I'm dealing with millions of documents.

Planning the indexing before implementing the application is a challenge 
unto itself.  My style seems to be writing one off scripts to test out the 
storage of indexes and do quite a bit of speed testing.  Doing so helps me 
see if there's a substantial performance degredation that will ultimately 
lead to a scaling disaster later on.

This article focuses on what temperature is best for a subset of cities in 
the `Geonames <http://www.geonames.org/>`_ free online database that will 
be used for geographic location and name reference.

The collection contains information on each city as well as the relevant 
counties, states, countries, and continents involved with each city in 
their own documents.  The administrative levels for each city take up a 
very small percentage of the collection.

Example Data
============

Only the relevant portions of the full documents are represented below.  
The most important bits are the normalized ``path`` key that is the ascii 
formatted version of the geographic areas name.

.. code-block:: javascript

    // Anchorage, Alaska (US) in db.places
    {
        "_id" : ObjectId("520415b8d3cdd67f514cd405"),
        "level" : "city"
        "name" : "Anchorage",
        "path" : "anchorage",
        "county" : "anchorage.municipality",
        "state" : "alaska",
        "country" : "us",
        "continent" : "na",
    }

    // The county in db.places
    {
        "_id" : ObjectId("5204144fd3cdd67f51d57c02"),
        "level" : "county",
        "name" : "Anchorage Municipality",
        "path" : "anchorage.municipality",
        "state" : "alaska",
        "country" : "us",
        "continent" : "na",
        "children" : {
            "cities" : [
                "anchorage",
                //...
            ]
        }
    }    

    // The state in db.places
    {
        "_id" : ObjectId("5204144ed3cdd67f51d4f913"),
        "level" : "state",
        "name" : "Alaska",
        "path" : "alaska",
        "country" : "us",
        "continent" : "na",
        "children" : {
            "cities" : [
                "anchorage",
                //...
            ],
            "counties" : [
                "anchorage.municipality",
                //...
            ]
        }
    }
    
    // The country in db.places
    {
        "_id" : ObjectId("5204144dd3cdd67f51d4eae2"),
        "level" : "country",
        "name" : "United States of America",
        "path" : "us",
        "continent" : "na",
        "children" : {
            "states" : [
                "alaska",
                //...
            ]
        },
    }

    // The continent in db.places
    {
        "_id" : ObjectId("5204144dd3cdd67f51d4e9f5"),     
        "level" : "continent",
        "name" : "North America",
        "path" : "na",
        "children" : {
            "countries" : [
                "us",
                //...
            ]
        }
    }

Too Hot
=======

Too Cold
========

Just Right
==========
