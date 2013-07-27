---
layout: post
title: "Keep Stuff In One Place"
tags: [postgresql]
---
{% include JB/setup %}

It's very useful to have your data in one place.

Explain a few stories about

* how we wanted Chartbeat to act a certain way. Couldn't because we didn't control that data.
* find out if people came back and purchased more after using a coupon.
* a/b tests difficult if not worrying only about one action

Let's look at the data for a typical web application:

* Main store for the business data. If an e-commerce site, products, orders, users, etc.
* Site analytics - information about page views, clicks, etc. Typically stored on Google Analytics.
* Logs (server, mail, web, etc)
* A/B tests. What group a user's in, what they saw, what they clicked, etc.
* User data in cookies
* Marketing e-mail campaigns (typically stored on Mailchimp)
* Background jobs for stuff to process asynchronously. Typically stored in redis, rabbitmq,
  Amazon's SQS, etc.
* Summary data for quick reporting. i.e. profit for for 2012 q3.

Mention scenarios for

* finding a path a user took to place an order
* lifetime customer value
* sessions vs users
* show page_views/sessions schema
* hstore for storing actions
* queue_classic, background workers in 9.3
* single backup for everything
* analytical sql queries
* getting too big? can move data to a tablespace or different db. postgresql_fdw for partitioning.
* amazon ebs allows for 1 terabyte per tablespace. but ebs sucks. :(
  m1.xlarge gives you 4x420 gig drives.

Disadvantages:

* Comes a point where it won't scale. But PG can do a lot efficiently. Might be more than you think.
* Single point of failure
* Historical data becomes too big? Partitioning can probably solve this.

You need to think about how much data you are storing and if it actually makes
sense to split everything up across different services. Doing so makes it
difficult to tie it back together. There are huge advantages to storing the
data about your business/system in a single place. PostgreSQL is an amazing way
to process and store all this data. You will have to learn SQL and relational
theory. That's not a bad thing.
