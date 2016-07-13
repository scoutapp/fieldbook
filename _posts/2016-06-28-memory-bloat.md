---
layout: post
title:  "Debugging memory bloat"
date:   2016-06-28 11:19:42 -0600
---

Rails memory issues are frequently more difficult - and more urgent - to resolve than performance problems: a slow Rails app may be painful, but if your app chews through all available memory on a host, the app is down.

This chapter shows how to identify memory-hungry controller-actions and specific memory-hungry requests, provides a visual representation of common pitfalls, and suggestions on fixing memory bloat.

### Memory bloat vs. memory leak

_Memory bloat_ is a sharp increase in memory usage due to the allocation of many objects. It's a more time-sensitive problem than a _memory leak_, which is a slow increase in memory usage and can be mitigated via scheduled restarts. 

Visually, here's the difference between bloat and a leak:

![bloat chart](/images/bloat_chart.png)

__A sharp increase in memory usage can bring down an application quickly__. A leak, by definition, won't trigger this behavior. 

While memory bloat can quickly cripple a site, it's actually easier to track down the root cause than a memory leak. __If your app is suffering from high memory usage, it's best to rule out a memory bloat given it's an easier problem to solve than a leak.__

### What goes up _doesn't_ come down

If one of your app's power users happens to trigger a slow SQL query, the impact is momentary. Performance will likely return to normal: it's rare for a slow query to trigger long-term poor performance.

If, however, that user happens to perform an action that triggers memory bloat, the increased memory usage will be present for the life of the Ruby process. While Ruby does release memory, it happens very slowely. 

It's best to think of your apps' memory usage as a high-water mark: __memory usage has no where to go but up.__

This behavior changes how you should debug a memory problem versus a performance issue.

### What you need to know about memory bloat

In order of importance:

1. __Memory Usage is a high-water mark__: Your Rails app will likely recover quickly when it serves a slow request: a single slow request doesn't have a long-lasting impact. This _is not the case_ for memory-hungry requests: just one allocation-heavy request will have a long-lasting impact on your Rail's app's memory usage.
2. __Memory bloat is frequently caused by power users__: controller-actions that work fine for most users will frequently buckle under the weight of power users. A single request that (a) renders the results of 1,000 ActiveRecord objects vs. 10, (2) processes a 10 MB image vs. a 500 KB image, or (3) handles a 1 MB payload vs. a 100 KB payload will trigger more allocations and have a long-term impact your app's memory usage.
2. __Focus on the _maximum_ number of allocations per controller-action__: a normally lightweight action that triggers a large number of allocations on a single request will have a significant impact on memory usage. Note how this is very different than optimizing CPU or database resources across an app.
3. __Allocations and memory increases aren't correlated on a long-running app__. Once your app's memory heap size has grown to accomodate a significant number of objects, a request that requires a large number of allocations won't neccisaliry trigger a memory increase. If the same request happened early in the Rails process' lifetime, it likely would trigger a memory increase.
4. __You will see a number of memory increases when a Rails application is started__: Ruby loads libaries dynamically, so many libaries won't be loaded until requests are processed. It's important to filter out these requests from your analysis.

### Using Scout to fix memory bloat

[Scout](https://scoutapp.com) can help you identify memory-hungry areas of your Rails app by:

1. Isolating the controller-actions generating the greatest percentage of allocations.
2. Viewing transactions traces of specific memory-hungry requests to isolate hotspots to specific areas of code.
3. Identifying users triggering memory bloat

#### Isolating allocation-heavy actions

If you're looking for a general understanding of which controller-actions are responsible for the greatest amount of memory bloat in your Rails app, a good starting point is the "Endpoints" section of Scout:

![endpoints](/images/endpoints.png)

Sort by the "% Object Allocations" column. This column represents the maximum number of allocations recorded for any single request for the given controller-action and timeframe. It's important to emphasis this isn't _mean allocations_. An example illustrates why _max allocations_ is needed to identify memory bloat. 

__Which endpoint in the chart below will result in the greatest memory usage?__

![endpoints vs](images/endpoints_vs.png)

Each ciricle represents a request from Endpoint A and Endpoint B. Analysis:

* Endpoint A has greater throughput
* Endpoint A averages more allocations per-request
* Endpoint A allocates far more objects, in total, over the time period

However, Endpoint B wins. The single allocation-heavy request from Endpoint B sets the high-water mark.

Back to Scout: from the list of endpoints, click on an endpoint to dive into the Endpoint Detail view. From here, you can click the "Allocations - Max" chart panel to view allocations over time.

SCREENSHOT

Beneath the overview chart, you'll see traces Scout captured over the current time period. Click the "Most Allocations" sort field from the pulldown. You'll see traces ordered from most to least allocations.

#### Reading a Scout memory trace

The screenshots below are generated from actual Scout transaction traces. A quick primer on the trace organization:

![trace_intro](images/trace_intro.png)

Method calls displayed in the trace details are organized from most to least allocations. The horizontal bar on the right visually represents the number of allocations associated with the method call(s) on the left. Some of the bars may have two shades of green: the lighter green represents the control case (what we view as a normal request) and the darker green represents the memory-hungry case.

#### Identifying users triggering memory bloat

It's common for memory bloat to be isolated to a specific set of users. Use Scout's [context api](http://help.apm.scoutapp.com/#adding-custom-context) to associate your app's `current_user` with each transaction trace if it's not easily identify from a trace url.

SCREENSHOT


### Common Pitfalls

Memory bloat often reveals itself in specific patterns - these patterns are illustrated via the Scout transaction trace below. 

#### ActiveRecord: rendering a large number of objects

When rendering the results of an ActiveRecord query that returns a large number of objects, __the majority of allocations frequently come from the view and not from instantiating the ActiveRecord objects__. When rendering the result of a large AR query, many Strings are allocated to represent each object's attributes as well as any templating around them (table rows and other HTML elements).

The screenshot below illustrates the difference between rendering a view with 1,000 records and one with ten. Two-thirds of allocations reside in the view:

![trace_intro](images/many_ar.png)

An example where this occurs:

{% highlight ruby %}
def employees
  @company = Company.find(params[:id])
  @company.employees
end
{% endhighlight %}

Fetching and rendering all employees for a company may work fine for the latest small startup, but it fall over for Apple, Inc.

The fix? Paginiation via [will_paginate](https://github.com/mislav/will_paginate) or [Kaminari](https://github.com/amatsuda/kaminari).

#### ActiveRecord: N+1 Database Queries

You probably already know N+1 queries are low-hanging fruit when it comes to speeding up your controller-actions. However, in addition to frequently being slower than a proper `includes`, they result in more allocations. 

The example below illustrates an N+1 when rendering out a list of 100 users and their associated company. Roughly 2x more allocations result from the N+1:

![trace_intro](images/n+1.png)

The steps to fixing N+1 queries are well-documented: the larger challenge is finding the worst offenders. [Scout can be used to identify the worst-offending N+1 queries](http://blog.scoutapp.com/articles/2016/04/28/stackprofin-to-n-1-query-detection) in your app.

#### ActiveRecord: selecting unused large columns

The standard ActiveRecord finder selects all columns from the respective table:

{% highlight ruby %}
User.all # User Load (756.6ms)  SELECT "users".* FROM "users"
{% endhighlight %}

If a table contains a large column (`binary` or `text`), there's a cost both in terms of memory usage and time if that column is returned. This is true even if the column is never accessed.

Identifying this scenario is more involved: __if a large column isn't accessed, it will not trigger additional Ruby allocations and will not appear in a memory trace__. Instead, look at the change in memory for that request:

![large col](images/large_column.png)

It's also likely the query may run slower as more data is read from the database and sent across the wire from the database to your app host: look for a slow query in the "Time Breakdown" section of the trace.

##### The fix

A couple possible approaches:

1. Only select what you need: `User.select(:name)`.
2. Move the large column to a dedicated table so the default finder is fast by default.

#### Uploading a large file

Your app will incur a significant memory increase to handle large file uploads. For example, the trace below illustrates the increase in memory usage when uploading a 1 GB file vs. a 1K file:

![trace_intro](images/large_file.png)

The majority of allocations occur unwrapping the file in the framework middleware. A common scenario where this behavior occurs: an app manipulates uploaded images that are typically 100-500 kB in size, but then a user attempts to upload a 10 MB image.

The workaround: send large files directly to a third party like S3. [See Heroku's docs on Direct to S3 Image Uploads in Rails](https://devcenter.heroku.com/articles/direct-to-s3-image-uploads-in-rails).

### Suggested Reading
