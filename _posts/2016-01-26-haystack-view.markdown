---
layout: post
title:  "Haystack and Multithreading"
date:   2016-01-26 15:36:01
categories: jekyll update
summary: 'In our current project, IsisCB Explore, we were experiencing a strange behavior lately:  sporadically while searching the search would return nothing, although after reloading the page there were actually quite a few results.'
author: Julia Damerow
project: Isis CB
---

In our current project, <a href="http://data.isiscb.org/">IsisCB Explore</a>, we were experiencing a strange behavior lately:  sporadically while searching the search would return nothing, although after reloading the page there were actually quite a few results. Moreover, the behavior didn't seem to reproduce on our local dev machines, but only on the server. Before getting further into the details, let me sketch out our system architecture real quick.

The system runs on Amazons Elastic Beanstalk using their load balancer. The application is built using the Django framework on top of a PostgreSQL database. For searching, we used Django's Haystack plugin with Elasticsearch. All pretty standard we thought.

In the end, it turned out that there were several things going on here. First of all, since we were using a load balancer we had to set our sessions to being sticky sessions. Once that was done though, we still experienced issues. Given that it worked fine on our dev machines, we started to eliminate possible causes such as the cache implementation (to make sure it isn't an issue with the session cache). Finally, after turning off the load balancing feature of Elastic Beanstalk we got the clue we needed. Instead of receiving empty results for queries that should yield results sets, we started to receive each others results, which told us that there must be some view objects that get shared between our two requests. This seemed very odd, as Django should create a new object per request. To make it short: turned out we implemented the Haystack search according to the documentation on the website, which suggested to use the FacetedSearchView as follows:

{% highlight python %}
from django.conf.urls.defaults import *
from haystack.forms import FacetedSearchForm
from haystack.views import FacetedSearchView

urlpatterns = patterns('haystack.views',
    url(r'^$', FacetedSearchView(form_class=FacetedSearchForm, facet_fields=['author']), name='haystack_search'),
)
{% endhighlight %}

As this is my first Django project I'm working on (I'm mostly a Java developer), I didn't question the example given in the documentation. I assumed that Django will create a new object for every request. Turns out this is wrong. For Django to do this, views have to inherit from generic views (`django.views.generic.View`). For Haystack this means that our views had to inherit from `haystack.generic_views` rather than `haystack.views`. Otherwise, there would be one view object for all request that would call the same form object on every request. This means in a multithreading environment, a parameter that was set in the view object (in our case the query string itself) could be overridden by a second request thread before the first request thread had a chance to retrieve the correct results. Once we rewrote the view class and changed the url patterns as shown below, the search seemed to work as expected. Our ``IsisSearchView`` now inherits from ``from haystack.generic_views import FacetedSearchView`` and is instantiated through its ``as_view`` method.

{% highlight python %}
url(r'^(?i)search/',
        IsisSearchView.as_view(form_class=MyFacetedSearchForm, queryset=sqs),
        name='haystack_search'),
{% endhighlight %}
