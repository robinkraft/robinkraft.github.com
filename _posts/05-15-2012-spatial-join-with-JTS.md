---
layout: post
title: Fires, and Spatial Joins with JTS
---

Introduction
============

I've got a soft spot for spatial joins and the Sample tool in Spatial Analyst for ArcGIS. This is partly because it's so darn useful and comes up all the time, but also raster sampling was one of the first GIS operation I coded up by hand, using Python and Numpy. Ah, those were the days ...

But today, we're going to take the easy route, and use libraries to do a spatial join with Clojure. Let's find out how many fires were detected in the US since late 2000. We need fires and a border polygon with its iso code.

Data
====

I've got some 46 million points representing fires detected by the MODIS sensors flying on NASA's Terra and Aqua satellites over the last 11 years. You can get more info and all the data from the [UMD FIRMS web site](http://firefly.geog.umd.edu/firms/). We'll be processing a 2.6gb textfile of fire points, and using polygon data from the [Global Administrative Areas](http://www.gadm.org/) database (GADM).

I happen to have the GADM data sitting on a [CartoDB](http://www.CartoDB.com) instance, so let's just grab some well-known-text in CSV format using the CartoDB SQL API:

    http://your-account.cartodb.com/api/v2/sql/?q=SELECT iso,ST_asEWKT(the_geom) AS geom FROM gadm0 WHERE iso='USA'&format=csv    

That was easy! You can grab a zipped copy [here](https://dl.dropbox.com/u/4448384/usa_poly.csv.zip) if you want     to follow along at home. I've stripped the `iso,geom` header line for convenience.

The file looks like this:

    USA,"SRID=4326;MULTIPOLYGON(((179.585615158081    
    52.0173091888428,179.652040481567 52.0247783660889 ...)))"

A line from the fires data looks like this:

    -18.735,144.626,324.8,1.1,1.0,05/13/2012,0030,T,78,5.0,297.2,24.9

A full description of the attributes is available on the [FIRMS site](http://firefly.geog.umd.edu/firms/faq.htm#attributes). We're only interested in the first two fields, latitude and longitude - the rest includes brightness, confidence, date, time, etc. You may have guessed that given the polygon defined above and all of these latlons, we should be able to do an intersection pretty easily. So let's get to it!

Setup
=====

If you're up for it, create a new Clojure project and add `[cljts "0.1.0"]` to your deps to get cljts and JTS. Check out [this post](http://robinkraft.github.com/2012/04/15/JTS-and-Clojure.html) for more details. Cljts pulls in the raw JTS library if you want to use it, but more importantly gives you some convenient helper functions. Thank you [sunng](https://github.com/sunng87)! Check out the [docs](http://sunng87.github.com/cljts/).

Basic processing
================

Cljts has a nice `read-wkt-str` function that we'll be using here. Let's set up the environment first:

{% highlight clojure %}
(use '[cljts.io :only (read-wkt-str)])
(use '[cljts.relation :only (intersects?)])
{% endhighlight %}

Then we load the polygon:

{% highlight clojure %}
(use '[cljts.io :only (read-wkt-str)])
(use '[cljts.relation :only (intersects?)])

(defn load-poly
  [path]
  (let [poly-str (slurp path)]
    (read-wkt-str poly-str)))

(def poly (load-poly "path-to-data"))
(envelope poly)
;=> 
{% endhighlight %}

Now we've got a JTS multipolygon object. Let's create a fire point based on a coordinate:

{% highlight clojure %}
(def pt (point (c 144.626 -18.735))) ;; note the xy order
(prn pt)

{% endhighlight %}

Now we can check intersection:

{% highlight clojure %}
(intersects? poly pt)
;=> false
{% endhighlight %}

That point happens to be in northeast Australia. So let's try another random latlon, from Yellowstone National Park:

{% highlight clojure %}
(intersects? poly (point (c -110.560913 44.801327)))
;=> true
{% endhighlight %}

{% highlight clojure %}
(defn spat-join 
  [poly-map point]
  (if (intersects? (:geom poly-map) (point)
      {:geom point :attributes (:attributes poly-map)})))

(def usa-poly-map {:geom poly :attributes {:iso "USA"}})
(def yellowstone-pt (point (c -110.560913 44.801327)))
(prn yellowstone-pt)
;=> #<Point POINT (-110.560913 44.801327)>

(spat-join poly-map pt)
;=> {:geom #<Point POINT (-110.560913 44.801327)> :attributes {:iso "USA"}}
{% endhighlight %}

It works! Now we just need to loop through 46 million fires points, storing the count by iso code, and then we would have a spatial join.

Performance
===========

So, looping through fire points line by line, intersecting each one with the polygon geometry in memory, would technically work. But it would take a while. On my 2010 iMac, intersect? returns `false` in as few as .0025 milliseconds. I suspect that `intersect?` first checks whether a point is within the bounding envelope of the polygon of interest - that operation screams. If the point doesn't fall within the bounding envelope, `intersects?` immediately returns false. If it does, I ended up waiting 268 milliseconds on average. Granted, the border of the US and its territories is a complex polygon, with 40+ thousand vertices, but I would love to see it faster.

I don't know enough (read anything) about computational geometry to know how hard it would be to do better than this. JTS [uses](http://tsusiatsoftware.net/jts/javadoc/com/vividsolutions/jts/algorithm/locate/package-summary.html) uses a simple polygon intersection algorithm to check for point location. [Tim Robertson](https://twitter.com/#!/timrobertson100), who [helped inspire](http://biodivertido.blogspot.com/2008/11/reproducing-spatial-joins-using-hadoop.html) this exploration, has a nice implementation of a [slab decomposition algorithm](http://en.wikipedia.org/wiki/Point_location#Slab_decomposition) that is incredibly fast. But as I said at the beginning, for now I want to stick with libraries, hoping to retain some semblance of a general solution.

Conclusions
===========

This exercise is going to be a problem. I happen to know (foreshadowing the next blog post) that of 46 million fires, 8.33 million fall within the USA's bounding envelope, which apparently stretches across most of the northern hemisphere (Alaska to Guam?).

![USA in GADM](http://gadm.org/data2/img/USA_adm.png)

So the question is, ignoring super-fast non-intersecting operations, how long should the exact intersection take?

    (/ (/ (/ (* 8.3 million points .268 seconds/point) 60) 60) 24.)
    ;=> 25.8 days

That's crazy! For completeness, note that ArcGIS chokes with a memory error after about 5 hours of work on a spatial join.

If only we could do this in parallel ...

I'm satisfied with the ease of using JTS in Clojure thanks to [cljts](https://github.com/sunng87/cljts), but this level of performance just won't do for the task at hand. Let's not forget that I've simplified the question. My real question - how many fires were detected in each country? - would take ages to complete given the size of the bounding envelopes of the US and Russia alone.

TL;DR
=====

[Cljts](https://github.com/sunng87) is a great way to use [JTS Topology Suite](http://tsusiatsoftware.net/) in Clojure, and has some convenient helper functions for IO and spatial relationships. But a naive loop through 46 million fires points to count up fires intersecting the US would take nearly 26 days using JTS' intersection algorithm. Look for a future post on doing this with Hadoop.