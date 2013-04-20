---
layout: post
title: "Data Driven Logic"
description: ""
category:
tags: [postgresql]
---
{% include JB/setup %}

Say packages 0-1 pounds should be shipped via USPS, 1-3 pounds should be shipped via FedEx, and more than 3 pounds via UPS.

Instead of:

    shipping_method =
      if shipping_weight < 1
        "USPS"
      elsif shipping_weight < 3
        "FedEx"
      else
        "UPS"
      end

Create a table containing the shipping method and the weights:

    create table shipping_methods (
      shipper text not null primary key,
      weight numrange not null,
      exclude using gist (weight with &&)
    );

    insert into shipping_methods values
      ('UPS', numrange(3, null)),
      ('FedEx', numrange(1, 3)),
      ('USPS', numrange(0, 1));

    select shipper from shipping_methods where 1.5 <@ weight_range;

     shipper
    ─────────
     FedEx

This uses [postgresql ranges](http://davisjeff.com/rangetypes4.pdf) plus an exclusion constraint to ensure that we don’t have overlapping weights entered.

Now, if the weights or shipping methods change, you don’t need to change the application code, you just modify the data in the shipping_methods table.

Now you can get the a list of the shipping method for each package pretty easily:

    select shipping_methods.shipper, packages.package_id
    from packages
    join shipping_methods on packages.weight <@ shipping_methods.weight
