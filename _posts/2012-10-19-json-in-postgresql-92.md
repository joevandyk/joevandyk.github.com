---
layout: post
title: "JSON in PostgreSQL 9.2"
description: ""
category:
tags: [postgresql, json]
---
{% include JB/setup %}

One of the big things coming in pg 9.2 is json support. This will allow you to return json representation of complex objects directly from the database.
Why should you care?  Imagine you have the following e-commerce schema:

    class Order
      has_many :products
      validates :email_address, :presence => true

      def description
        "".tap do |result|
          result << "#{ email_address } purchased the following items:\n"
          line_items.each do |item|
            result << " * #{ quantity } x #{ item.product.name }"
            result << " * #{ item.product.image.url }"
          end
        end
      end
    end

    class LineItem
      belongs_to :product
      validates :quantity, :presence => true
    end

    class Product
      has_many :line_items
      has_one :product_image
    end

    class ProductImage
      belongs_to :product
    end

You want a report page that displays who purchased what.

    Order.all.each do |order|
      puts order.description
    end

This will get each order, then retrieve the line items, then retrieve the products, then retrieve the images. A zillion sql queries.

So you use Eager Loading to reduce the n * n1 * n2 * n3 + 1 query problem.

    Order.includes(:line_items => { :product => :product_image }).each do |order|
      puts order.description
    end

This is better, but we’re still doing four queries (one each for Order, LineItem, Product, and ProductImage).
(In a real e-commerce application, there’s a lot more tables involved for reports like these.)
Plus the API sucks, you have to remember to specify the includes every place you want to process orders,
line items, products, and images — if you forget, you will be executing a lot of queries.

Wouldn’t it be nice to get all the information back in one sql query, without worrying about eager loading?
And have this information be available to every application that accesses your database?

Enter postgresql’s json support.

This is the json we want returned:

    [
     {  id: 1,
         email_address: 'joe@tanga.com',
         line_items: [
           { quantity: 2, product: { id: 1, name: 'Product Name', image: { url: '/url.png'}}},
           { quantity: 3, product: { id: 2, name: 'Product Name', image: { url: '/url.png'}}}]
      }

      {  id: 2,
         email_address: 'another@tanga.com',
         line_items: [ ... ]
      }
    ]

This [gist](https://raw.github.com/gist/3021435/6158b38673b7168ad7666744d740ac9f82f19304/gistfile1.txt)
shows my first attempt.
