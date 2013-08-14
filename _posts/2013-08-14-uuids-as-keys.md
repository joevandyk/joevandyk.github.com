---
layout: post
title: "Use UUIDs as Keys"
description: ""
category:
tags: [postgresql]
---
{% include JB/setup %}

If data integrity is critical for your systems, you should be using UUIDs for keys.
True, they take up a bit more space in storage and memory and aren't quite as fast.
But since UUIDs are by definition unique, your data is more likely to be consistent.

An example:

    begin;
    create table products (product_id serial primary key);
    create table orders   (order_id serial primary key,
                           product_id integer references products not null);

    insert into products values (default), (default);
    insert into orders (product_id)  select product_id from products;
    insert into orders (product_id)  select order_id from orders; -- OOPS! This worked! Should've failed!

We accidently inserted bad data into the orders table in that second insert. We should've
been selecting from the `products` table, not the `orders` table. But because 
the primary keys of `products` and `orders` share the same values, 
postgresql didn't complain and let us insert incorrect data.

Let's fix this.

    begin;
    create extension "uuid-ossp";
    create table products (product_id  uuid primary key default uuid_generate_v4());
    create table orders   (order_id    uuid primary key default uuid_generate_v4(),
                          product_id   uuid references products not null);

    insert into products values (default), (default);
    insert into orders (product_id)  select product_id from products;
    insert into orders (product_id)  select order_id from orders;  -- HOORAY! This failed to insert.

    -- The above insert fails with:
    -- ERROR:  insert or update on table "orders" violates foreign key constraint "orders_product_id_fkey"
    -- DETAIL:  Key (product_id)=(a593fdde-e172-4546-8e7a-174a5bd9d94b) is not present in table "products".

Now we aren't allowed to insert an incorrect `product_id` into the `orders` table.

For more information on UUIDs in postgres:

* <http://www.postgresql.org/docs/9.2/static/datatype-uuid.html>
* <http://www.postgresql.org/docs/9.2/static/uuid-ossp.html>
