---
layout: post
title: Spatial joins with Hadoop and Cascalog
---
<p class="meta"> 27 May 2012 - San Francisco</p>

Spatial joins with Hadoop and Cascalog
======================================

[Last
time](http://robinkraft.github.com/2012/05/26/fires-spatial-join-Clojure-JTS.html) we looked at a simple spatial join of a single latlon with a polygon geometry of the United States. `(intersects? poly pt)` works like a charm, but it takes a while to complete for the detailed US polygon geometry. Checking millions of points would take weeks at .27 seconds per point.

But! We have Hadoop! This embarassingly parallel problem would be much more tractable if expressed as map-reduce jobs. We'll let [Cascalog](http://github.com/nathanmarz/cascalog) handle mapping and reducing, I just have to write a few queries.

The query
-----------

 Full code is [available on Github](https://github.com/robinkraft/spatialog/blob/develop/src/clj/spatialog/easyjoin.clj). Here's the main query:

{% highlight clojure %}
(defn join-n-count
  [poly-path points-path intersect-op]
  (let [[iso poly-geom] (import-poly poly-path)
        pts-tap (points-tap points-path)]
    (<- [?count ?iso]
        (pts-tap ?lat ?lon)
        (add-fields iso :> ?iso)
        (intersect-op [poly-geom] ?lat ?lon)
        (c/count ?count))))
{% endhighlight %}

`join-n-count` has two interesting features.

First, and perhaps most importantly, it loads the polygon geometry before the body of the query. `import-poly` is another Cascalog query that prepares the polygon geometry and extracts the iso code, returning both as Clojure data structures using the `??-` operator. Because we execute the query and load the data structure inside the let statement, we can pass it as an argument to the [parameterized](http://nathanmarz.com/blog/news-feed-in-38-lines-of-code-using-cascalog.html) (see section two) `intersect-op` operation.

This mainly means that each machine gets its own copy of the data structure, serialized to the job conf. Limits on the size of the job conf means this won't work with large data structures. An alternative method would use a dreaded `cross-join` to intersect each point with the polygon, but that runs through a single reducer.

Second, the `intersect-op` argument to `join-n-count` expects a parameterized [custom operation](https://github.com/nathanmarz/cascalog/wiki/Guide-to-custom-operations) for intersection. This makes it easy to simply count fires falling inside the US polygon envelope (or bounding box), then do a full intersection. The query itself doesn't change, but boy the run times sure do! Envelope intersects are much faster, given the simple (rectangular) polygon geometry.

{% highlight clojure }%
(deffilterop [intersects-op [poly]] [lat lon]
  "Parameterized intersect"
  (let [pt (latlon->point lat lon)]
    (intersects? poly pt)))

(deffilterop [intersects-env-op [poly]] [lat lon]
  "Parameterized intersect only checks envelope intersect"
  (let [pt (latlon->point lat lon)]
    (intersects? (envelope poly) pt)))
{% endhighlight %}

The main query has three parts. First, it loads the polygon boundary, then loads all the fires points in the map stage, performing the intersect operation on the reduce side. Each reducer emits a tuple of the number of fires it saw that fell inside the polygon (or its envelope). The final step combines these separate results to give a final total of fires inside the US polygon or its envelope.

Test environment
------------------

I'm using [Elastic MapReduce](http://aws.amazon.com/elasticmapreduce/) from Amazon Web Services, using m2.4xlarge spot instances with 68.4gb of RAM and 8 virtual cores. This works out to 30 mappers and 24 reducers per machine. I'm working with a 1-machine "cluster" and a 10-machine cluster. Set up is simple:

    git clone git@github.com:robinkraft/spatialog.git
    cd spatialog
    lein uberjar

    zip -d spatialog-0.1.0-SNAPSHOT-standalone.jar org/apache/xerces/\*
    zip -d spatialog-0.1.0-SNAPSHOT-standalone.jar META-INF/services/\*

    screen -Lm hadoop jar spatialog-0.1.0-SNAPSHOT-standalone.jar clojure.main

From the REPL:

{% highlight clojure %}
(use 'spatialog.easyjoin)
(in-ns 'spatialog.easyjoin)
;; sample query
(??- (parameterized-join "s3n://formaresults/test/gadm/usa.csv" "s3n://formaresults/test/firesnoheader/firesnoheader" intersects-env-op))
{% endhighlight %}

The lines above deleting files from the jar file are my workaround to an [issue](https://www.google.com/search?sugexp=chrome,mod=5&sourceid=chrome&ie=UTF-8&q=jts+xerces+version) with JTS's dependency on an old version of [Apache Xerces](http://xerces.apache.org/). This causes Cascalog jobs to fail for reasons I have not yet uncovered, but all is not lost! If you're working locally, just delete `xercesImpl-2.4.0.jar` file from the `lib` folder in your project. Some day I hope to understand why Xerces causes problems. [Stacktrace here](https://gist.github.com/2802301).

Results
---------

Of 46 million fires detected from 2000-2012 using the MODIS sensors on Terra and Aqua, 8,331,135 fell inside the bounding envelope of the US border polygon. Of those, 950,055 were detected actually detected within the border of the United States.

Performance
-------------

I ran three queries on each cluster on two separate fires datasets, one with just 2700 lines, the other with the full 46 million. Take the exact times with a grain of salt, rather focus on the orders of magnitude. The first run, the envelope intersect on the small dataset and small cluster, established a baseline of 8 min. Running the exact intersect on the polygon for these 2700 fires took 9 min.

The 1-machine cluster `C1` took 11 min. for the envelope intersect for the 46 million fires - not meaningfully different from the 2700 fires above. The 10-machine cluster `C10` took 13 min. I chalk the longer time for the larger cluster up to the [small files problem](http://www.cloudera.com/blog/2009/02/the-small-files-problem/). Hadoop is designed to power through massive files, not tiny ones, so performance suffers.

The interesting times are for the exact intersect for all fires. `C1` never completed because I got tired of waiting. `C10` ended up finishing in 5h15, compared to about 13 min. for the envelope intersect. We can infer that `C1` likely would have finished in upwards of 50 hours. In total, `C10` used 17.5 CPU days for this job, about 1/3 less than I had expected given the performance of my iMac. The total cost of EC2 time was $45, with another $25 for the EMR service.

Discussion
------------

All in all, I'm pleased that this finally worked, although the 17.5 CPU days it took seems excessive. But since I had no other viable options I can't complain. The complexity of the polygon (40+ thousand vertices) was the main contributor to the 5h15 it took to complete on the larger cluster. An otherwise identical query intersecting polygon envelopes took only 13 minutes, or 25x less time. Were it not for the small size of the files that add to Hadoop overhead, I would expect the envelope query to approach the 100x less time it takes to do the envelope intersection at the REPL. For larger files, Hadoop's IO model really gets fast, but for small files it never gets a chance to ramp up.

This Hadoop job only does a small part of what I'd like to do: count up fires for all countries. But the run time for what I have done makes it seem infeasible to do the join for all countries without moving to highly customized workflows and custom code. Then again, I didn't try building a spatial index, something that could make this more feasible.

As it stands, the success of this exploration of spatial joins rests on the artificial simplicity of the question. But I am convinced that between Hadoop distributed caches, simplified polygons, and some kind of indexing, it will be possible to count up fires by country without spending too much or waiting months for the job to finish.
