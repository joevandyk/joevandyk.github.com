---
layout: post
title: "Store Page Views in PostgreSQL"
description: ""
category:
tags: [postgresql]
---
{% include JB/setup %}

Really rough draft follows.

If you are running a web application, you should be storing user sessions and page views in a database. PostgreSQL
has good support for this sort of thing (rewrite this clumsy sentence).

                               Table "public.sessions"
       Column   |           Type           |              Modifiers
    ------------+--------------------------+-------------------------------------
     session_id | uuid                     | not null default uuid_generate_v4()
     created_at | timestamp with time zone | not null default now()

    Indexes:
        "sessions_pkey" PRIMARY KEY, btree (session_id)
        "sessions_created_at_idx" btree (created_at)

Every user gets a session_id assigned to them. This is stored in a cookie. 

                                                 Table "analytics.page_views"
        Column    |           Type           |                                  Modifiers
    --------------+--------------------------+-----------------------------------------------------------------------------
     page_view_id | bigint                   | not null default nextval('analytics.page_views_page_view_id_seq'::regclass)
     session_id   | uuid                     | not null
     created_at   | timestamp with time zone | not null default now()
     site_id      | integer                  | not null
     query_string | hstore                   | default ''::hstore
     path         | citext                   | not null
     user_agent   | citext                   | not null
     referral_url | text                     |
     ip_address   | ip4                      |
     user_id      | integer                  |
     details      | hstore                   |
     http_method  | analytics.http_method    |

    Indexes:
        "page_views_pkey" PRIMARY KEY, btree (page_view_id)
        "page_views_created_at_idx" btree (created_at)
        "page_views_ip_address_idx" btree (ip_address)
        "page_views_session_id_idx" btree (session_id)
        "page_views_user_id_idx" btree (user_id)

    Foreign-key constraints:
        "page_views_session_id_fkey" FOREIGN KEY (session_id) REFERENCES sessions(session_id)
        "page_views_site_id_fkey" FOREIGN KEY (site_id) REFERENCES sites(id)
        "page_views_user_id_fkey" FOREIGN KEY (user_id) REFERENCES users(id)


Every page view is stored in this table. The page view is inserted after the page is generated.
In Rails, we do this in an after_filter in the controller. 

page_views has a foreign key to the sessions table, so we can associated user sessions to page_views.
We run multiple sites off the same database, so there's also a reference to the sites table.

If the user is logged in, we store the user's id for each page view.
At first, I thought the user_id should be stored in the sessions table, but that wouldn't work 
as we need to track a session that goes from being a guest user to a logged in user. They would keep
the same session, but their page views before logging in wouldn't have a user_id associated with them.

You could store the full URL visited, but I chose to break the URL down into path (stored as a citext)
and a hstore representing the query string. I don't need to store the http schema (http vs https) or the 
domain, but you could do that if needed.

We track user agents, referral url, and the http method as well. The http method is stored
as an enum that contains all the http methods (get, post, options, etc). 

We also use the details hstore to store arbitary information about the page view. For example, 
if they are viewing a product, we store 'product_id' => '3893'. 

When someone places an order, we store a reference to the current session_id.
Since we store the session_id for each of the page views, we can then track the complete path
a user took to placing an order.

We use a database function for inserting page views. It looks similar to:

    CREATE OR REPLACE FUNCTION analytics.record_page_view(hstore)
     RETURNS uuid
     LANGUAGE plpgsql
    AS $function$
    <<locals>>
    declare
      session_id   uuid   := $1->'session_id';
      site_id      int    := $1->'site_id';
      user_agent   citext := $1->'user_agent';
      path         citext := $1->'path';
      query_string hstore := $1->'query_string';
      user_id      int    := $1->'user_id';
      ip_address   ip4    := $1->'ip_address';
      details      hstore := $1->'details';
      http_method  analytics.http_method := $1->'http_method';
      affiliate_network          text := $1->'affiliate_network';
      affiliate_name             text := $1->'affiliate_name';
      affiliate_network_click_id text := $1->'affiliate_network_click_id';
      referral_url               text := $1->'referral_url';
      new_page_view_id bigint;
    begin
      -- If there's no session_id or they passed in a session_id that we don't recognize, 
      -- make a new session and return the id into the session_id variable.
      if session_id is null or (select count(*) from sessions where sessions.session_id = locals.session_id) = 0 then
        insert into sessions (session_id) values (default) returning sessions.session_id into session_id;
      end if;

      -- insert stuff into page views. not a fan how verbose sql is for inserting lots of columns.
      insert into analytics.page_views
        (session_id, http_method, details, user_id, ip_address, site_id, user_agent, path, query_string, referral_url)
        values (session_id, http_method, details, user_id, ip_address, site_id, user_agent, path, query_string, referral_url)
        returning page_view_id into new_page_view_id;

      -- Some of our visits come from affiliates. If the affiliate_network parameter is present, record that information
      -- as well. 
      if affiliate_network is not null then
        insert into analytics.affiliate_network_visits (page_view_id, affiliate_network_click_id, affiliate_name, affiliate_network)
        values (new_page_view_id, affiliate_network_click_id, affiliate_name, affiliate_network);
      end if;

      -- Return the session_id.
      return session_id;
    end $function$;


Once PostgreSQL 9.3 comes out, the function will be changed to accept JSON instead of an hstore as it's
awkward to construct an hstore that contains another hstore (details and query strings).
We pass an hstore to the function so the function signature doesn't have to change every time we 
add or remove an argument.

Every page view calls the above function. Only one function is called, even if we need inserts to multiple tables.
This is done to reduce the amount of SQL queries that need to happen.

We can pass additional information into the record_page_view function. For example, we want to store
information about what affiliate sent us traffic. 
That information is stored into the analytics.affiliate_network_visits table.

Storing user actions and page views in PostgreSQL is one of the best things I've done for our business.
Examples of how we use it:

* When an order is placed, we sometimes need to credit affiliates. It's simpler to look up in the database
what affiliate sent us the customer and when they sent us the customer. If the visit from the affiliate
was recorded in the last 30 days (this number can vary depending on the particular affiliate), then we 
credit the affiliate. 
* We can find the complete journey a user took to place an order.
* We can get real-time data on activity on our site. We can setup alerts if there's unusual activity.
Sometimes if a product is super popular, we want to know where that traffic is coming from.
* This opens the way for AB testing. Sessions can be put into buckets and then it's simple to 
write SQL queries to figure out which bucket performed best.
* `select count(*), user_id from analytics.page_views where user_id is not null and created_at > 'today' order by 1 desc limit 50;`
Which of our users are especially active today?
* `select count(*), details->'product_id', p.name from analytics.page_views join products p on p.id = (details->'product_id')::integer where created_at > 'today' group by details->'product_id' order by 1 desc limit 30;`
What products are most viewed today?

You can go absolutely nuts with analytical queries. It's *incredibly* useful to join your app's data
against page view data. Most people use Google Analytics, but it can be a pain to join GA's data
against yours. And GA prevents you from getting at specific user's data -- it's against GA's terms 
of service to store unique user identifiers. If a user writes in saying they are having a problem buying
something, you can't go to GA and look at what they did on the site. Good luck trying to figure out
how much revenue *and* profit you've received from an affiliate via GA. Plus, most 3rd party analytic
systems require javascript.

Of course, there are downsides to this. The biggest one is performance and storage costs. If you have a
really busy site, storing all the page view data can become cumbersome. But this isn't a problem for 99.9%
of sites.  But PostgreSQL 9.3 will help here! [postgres_fdw](http://www.postgresql.org/docs/9.3/static/postgres-fdw.html)
allows you to insert and query other databases from inside PostgreSQL. You could store page views and user
actions on one or more other databases which should solve the scalability issue.

A busy site could also record page views info to a logfile that could be inserted into the database offline.
I don't need that complexity at this point and probably won't for a long time, if ever. 

Another issue is handling robots. Robot hits are also recorded in the database. You can solve 
this by having a table that lists robot user agents and either deleting those page views or 
moving them to a separate table periodically. 
(As an aside, I'm looking forward to PostgreSQL's 
[custom background workers](http://www.postgresql.org/docs/9.3/static/bgworker.html)
that will let me have the database call a function every minute, hour, etc. This removes the 
need for a cron-like app setup that's responsible for calling the robot page view cleanup function
periodically).

FWIW, storing 5 weeks of page views takes up 3.5 gigabytes for us (including indexes). I figure I have
about a year before I'd need to start worrying about splitting the page view data into a separate database, 
assuming our traffic keeps increasing at the current rate.
