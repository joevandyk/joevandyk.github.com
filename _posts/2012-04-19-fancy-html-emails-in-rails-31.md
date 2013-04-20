---
layout: post
title: "Fancy HTML Emails in Rails 3.1"
description: ""
category:
tags: [email, rails]
---
{% include JB/setup %}

Fancy HTML Emails with Rails 3.1
Getting HTML emails to look nice is a pain.  Most email clients can’t use stylesheets,
so you have to embed all the styles inline in the HTML.  You also have to write a separate
plain-text version of the email.  And popular email clients (Outlook, Windows Live Mail, etc)
render html email using some very weird rules.

Here’s what our order email looks like in Gmail:

Here’s what it looks like on the iPhone:

Not too shabby.

We found this [great article](http://webdesignerwall.com/general/make-your-html-email-5-times-more-mobile-friendly)
on how to make mobile email great looking:


We also used the [Premailer](https://github.com/alexdunae/premailer) gem to automatically
inline the linked stylesheet in the email views.

Our email layout looks something like:

    %html
      %head
        = stylesheet_link_tag 'email'

        %style{:type => "text/css"}
          :sass
            @media all and (max-width: 480px)
              table#container
                width: auto !important
                max-width: 600px !important
             ... and so on for the mobile code

      %body
        Email body here.
        %table
          Lots of tables.

We include a stylesheet in the HTML. Premailer downloads it, processes it, and inserts the css rules inline in the HTML.

The @media rules need to be inline in the email layout, since Premailer can’t handle those being in a separate css file yet.

We use premailer-rails3 to integrate Premailer into Rails 3.  Unfortunately, we found a bunch of bugs in premailer and premailer-rails3.
So we forked [premailer](https://github.com/joevandyk/premailer) and
[premailer-rails3](https://github.com/joevandyk/premailer-rails3).  The forks fix some encoding bugs, remove some
weird css processing stuff done by premailer-rails3, allow premailer to not strip out
embedded `<style>` rules in the email layouts, and some other things.

We also found a bug in sass-rails, where you can’t embed image-urls in inline sass code.  See
[https://github.com/rails/sass-rails/issues/71](https://github.com/rails/sass-rails/issues/71)

Premailer-rails3 hooks into ActionMailer when the email actually being delivered, not just generated.
When running tests, email is not actually sent, so the premailer-rails3 hooks don’t get ran during tests.
I haven’t spent the time to see if it’s possible to get the premailer processing to run during tests, but that would be a nice thing to do.

Also, our forks on premailer-rails3 assume that you want premailer to go out and actually
download the linked CSS files.  It should be possible to use the Rails 3.1 asset pipeline
to get the processed css without downloading it.

A very special thanks goes to [Jordan Isip](http://www.jordanisip.com/) who did the super annoying job of making sure the emails
look great in all the different clients out there.  Writing that CSS/HTML did not look fun.
