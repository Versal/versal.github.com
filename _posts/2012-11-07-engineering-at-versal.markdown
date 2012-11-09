---
layout: post
title: "Engineering at Versal"
date: 2012-11-07 17:38
comments: true
categories: frameworks
---

There are many choices for implementing RESTful API architectures in Scala.  Some of us here at Versal had a good experience with Play 2.0 and Swagger, so initially we went that route.  However, Play configuration has quickly become unwieldy.  As we use CoffeeScript on the front end, we didn't need any of the templating functionality of Play, and we also wondered whether Play is as performant as it claims to be.

<!--more-->

The engineering spirit of Versal demands that any assumption be tested.  So for the most performant web framework, we went ahead and tested a whole bunch of them.  The results are available as the [Scamper](http://github.com/Versal/scamper) project.  They are quite surprising!