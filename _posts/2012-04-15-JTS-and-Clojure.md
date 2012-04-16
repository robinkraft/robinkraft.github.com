---
layout: post
title: JTS Topology Suite and Clojure
---

This is the first in a series of posts about using the [JTS Topology Suite](http://tsusiatsoftware.net/jts/main.html) (JTS) in Clojure, particularly in conjunction with Cascalog and Hadoop. To start I'll briefly look at the setup needed for using JTS with Clojure. Further along I'll explore replicating GIS operations using Cascalog/Hadoop.

JTS Topology Suite is "an API for modelling and manipulating 2-dimensional linear geometry". That really just means you can use it to do typical GIS operations, like intersecting polygons or determining whether a line touches a polygon, among many other things.

Written in Java, JTS should work directly with [Clojure](http://www.clojure.org), but there are already at least three partial wrappers for JTS that provide greater convenience.
* [jts-clj](https://github.com/jsofra/clj-jts) provides a Clojure-ish API for creating JTS geometry instances.
* [cljts](https://github.com/sunng87/cljts)'s geometry API feels like less idiomatic Clojure at first glance, but it does wrap some of JTS' spatial analysis and spatial relationship functions.
* [geoscript-clj](https://github.com/iwillig/geoscript-clj) aims more broadly at working with geospatial data in Clojure, using GeoTools and JTS.

I haven't worked with these enough to pick a favorite, but I started with cljts because of the spatial functions and because it seems simpler than geoscript-clj. I'll probably revisit this at some point.

To get started with cljts, add `[cljts "0.1.0"]` to your `project.clj` file. Then you can create a simple coordinate geometry:

{% highlight clojure %}
user> (use 'cljts.geom)
user> (c 10 20)
#<Coordinate (10.0, 20.0, NaN)>
{% endhighlight %}

And there you have it. Create a polygon by stringing together coordinates in a vector, being careful to close the polygon by making the first and last coordinates the same. Note that `polygon` expects two rings - the external border and a hole - but you can create a polygon without a hole by replacing the hole linear-string with `nil`:

{% highlight clojure %}
user> (polygon (linear-ring [(c 20 40) (c 20 46) (c 34 56) (c 20 40)]) nil)
#<Polygon POLYGON ((20 40, 20 46, 34 56, 20 40))>
{% endhighlight %}

cljts provides an interface to JTS' [DE9-IM](http://en.wikipedia.org/wiki/DE-9IM) spatial relationship methods. You can check intersection using `intersects?`.

{% highlight clojure %}
user> (def poly (polygon (linear-ring [(c 20 40) (c 20 46) (c 34 56) (c 20 40)]) nil))
user> (def pt (point (c 21 45)))
user> (intersects? pt poly)
true
{% endhighlight %}

Other functions for determining spatial relationships are described in the `cljts.relation` [documentation](http://sunng87.github.com/cljts).