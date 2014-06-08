# A performance shootout for RESTful libraries

*November 7, 2012*

*James*

One of our early projects at Versal was building a thin RESTful API on top of our Scala-based platform.  This would allow access to backend services by rich frontend applications, and establish conventions for data and service contracts among our systems.

Initially we went with a Play 2.0 and Swagger implementation, but its configuration felt unweildy, and the bulk of its feature set was unnecessary for just a simple RESTful API.  At this point we stepped back to survey the landscape.

Some of us had experience with RESTful Java libraries such as Jersey, Spring MVC, Struts, etc., while others had used Scala libraries such as Scalatra and Spray.  It was apparent that we needed to cast a wide net and compare what we found, so we created the [Scamper](http://github.com/Versal/scamper) project.

Scamper pits several libraries against each other:

![Fast Test](https://raw.github.com/Versal/scamper/master/readme/fast-test.png)

We made our best guesses to configure each project in a comparable way, and designed both non-blocking "fast" tests and blocking "slow" tests for each using both ApacheBench and JMeter.

Based on initial analysis, the fastest implementation uses raw Servlet 3.0.  Play 2.0 was somewhat disappointing, but Scalatra turns out to be a nice tradeoff between performance and implementation simplicity.  We decided to go with Scalatra.

We have had good input from the community on how to optimize Play 2.0, yielding significant performance improvements.  We look forward to learning more about each approach!
