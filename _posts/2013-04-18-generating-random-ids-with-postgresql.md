---
layout: post
title: "Generating Random Primary Keys with PostgreSQL"
description: "How to do this"
category: 
tags: ["postgresql"]
---
{% include JB/setup %}

It's sometimes nice to have random IDs for primary keys, so instead of

    id | username
    -----------------
    1  | joevandyk
    2  | murdoch

You would have

    id | username
    ------------------------
    1383892921 | joevandyk
    2393939114 | murdoch

You could use UUIDs, but that takes up additional storage space and can make URLs look funny.
And you might need numeric IDs for some reason.
I've found the best way to do it is to define the following function:

    CREATE OR REPLACE FUNCTION pseudo_encrypt(VALUE int) returns bigint AS $$
    DECLARE
    l1 int;
    l2 int;
    r1 int;
    r2 int;
    i int:=0;
    BEGIN
     l1:= (VALUE >> 16) & 65535;
     r1:= VALUE & 65535;
     WHILE i < 3 LOOP
       l2 := r1;
       r2 := l1 # ((((1366.0 * r1 + 150889) % 714025) / 714025.0) * 32767)::int;
       l1 := l2;
       r1 := r2;
       i := i + 1;
     END LOOP;
     RETURN ((l1::bigint << 16) + r1);
    END;
    $$ LANGUAGE plpgsql strict immutable;

    select pseudo_encrypt(1); => 1241588087
    select pseudo_encrypt(1); => 1241588087
    select pseudo_encrypt(2); => 1500453386
    select pseudo_encrypt(2); => 1500453386
    select pseudo_encrypt(484839329); => 7423122

This is known as a [Feistel Cipher](http://en.wikipedia.org/wiki/Feistel_cipher).
As you can see, given the same input, it'll create the same output. And there is 
a 1-1 ratio of input to output.

This means you can use a [sequence](http://www.postgresql.org/docs/9.2/static/sql-createsequence.html)
combined with this function to get somewhat randomized IDs. 

<i>To be continued...</i>
