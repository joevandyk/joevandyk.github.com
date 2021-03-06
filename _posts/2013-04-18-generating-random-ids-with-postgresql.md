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
And UUIDs are a bitch to type in. And they make barcodes long.

Randomized IDs are nice because it eliminate mistypes -- say you have 10,000 SKUs in a system
and someone is typing in a SKU ID. If the codes are sequential, it's going to be easy to to
accidently enter the wrong ID, "1234" vs "1324". If the IDs are randomized with lots of space
between the generated numbers, this problem largely goes away (at least, until you have more than
a million SKUs).

I've found the best way to do it is to define the following function:

    -- Taken from http://wiki.postgresql.org/wiki/Pseudo_encrypt
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
    select pseudo_encrypt(484839329); => 7423122

This is known as a [Feistel Cipher](http://en.wikipedia.org/wiki/Feistel_cipher).
As you can see, given the same input, it'll create the same output. And there is
a 1:1 ratio of input to output.

It only supports 32-bit input. I'm sure you could adjust it for 64-bit numbers, but that's more than
I need at the moment.

This means you can use a [sequence](http://www.postgresql.org/docs/9.2/static/sql-createsequence.html)
combined with this function to get randomized, guaranteed-unique IDs.

    -- Create a sequence for generating the input to the pseudo_encrypt function
    create sequence random_int_seq;

    -- A function that increments the sequence above and generates a random integer
    create function make_random_id() returns bigint as $$
      select pseudo_encrypt(nextval('random_int_seq')::int)
    $$ language sql;

    create table f (
      -- the id column now has a random default ID
      id integer primary key default make_random_id()
    );

    insert into f values (default);
    insert into f values (default);

    select * from f;

         id
    ────────────
     1241588087
     1500453386


Coming next: How to generate small unique random strings suitable for primary keys.
