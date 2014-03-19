---
layout: post
title: When to use http session affinity in GLASS
description: ""
category: null
tags: []
published: true
---

{% include JB/setup %}
[Seaside](http://www.seaside.st/) applications that deploy in [GLASS](http://gemstonesoup.wordpress.com/category/glass/) will almost always run using multiple Gemstone VMs to put the parallel request processing capability of the platform to use. The standard GLASS setup of 3 VMs already brings you a long way in the scaling of your web application. Because of the way Seaside works in GLASS, incoming requests can be freely distributed to any VM, regardless of the Seaside session they are related to. This makes load balancing in GLASS easy, without the need to [coordinate request handling](http://book.seaside.st/book/advanced/deployment/deployment-apache/mod-proxy-balancer) or install any kind of session affinity.

Although I think this is a brilliant achievement, the way this works in GLASS can cause some performance problems when you are not careful with the way your application triggers requests. In this post, I will explain when and why this happens and how we solved it (this post's title might already give you a clue ;-).

In short **performance will degrade when your web application triggers multiple AJAX requests to the server concurrently**. This is because the standard load balancer's behavior interferes with the way GLASS handles concurrent requests for a single Seaside session. The load balancer will distribute the concurrent requests over the different VMs (which blocks them from processing requests for other sessions) but because requests for a single Seaside session cannot be processed in parallel, the processing of each such ajax request will block the VM for a longer time period than necessary. Finally, the total time for all requests to finish will be longer than when the requests would have been processed sequentially by a single VM.

The reason for this behavior is that GLASS locks the WASession instance when processing a request for that Seaside session. This ensures that no other request for the same Seaside session can be processed by a concurrent thread (VM). In this post:[GLASS 101: Simple&nbsp;Persistence](http://gemstonesoup.wordpress.com/2008/03/09/glass-101-simple-persistence/), Dale explains us what happens when such a lock is denied:

> So, when an object lock is denied while processing a request, we throw an exception, abort the transaction, delay for a bit and retry the request. Each request is handled in its own thread, so while the request is delayed, other requests can be processed by the server. A nice, clean solution that fits very well within the GemStone/Seaside framework.

This means that the request handling will be delayed for some time and then retried. Depending on the processing time of each request and the number of concurrent requests, this can cause quite some delay. For example, in [Yesplan](http://www.yesplan.be/), we noticed how a particular request took more than three times longer to complete when we activated 3 VMs instead of a single one. (Some more explanation of http request retries can be found in this post of Dale:[GLASS 101:...Fire](http://gemstonesoup.wordpress.com/2008/03/17/glass-101-fire/) 

With the increased interactivity we are expecting from web applications, it's not uncommon to end up with multiple ajax requests being fired in parallel to your server. I would even say more: it's the desired behavior of an asynchronous request that you can send other such requests while 'waiting' for a response. Although it's obviously a good idea to maximize the bundling of ajax requests, this is not always desirable from an implementation point-of-view (more on that in another post). And whenever performance-related code changes start interfering with code quality or complexity, we should give them a serious thought.

A solution to the problem that does not interfere with how we implement the application and its request handling is by configuring the http load balancer to perform session affinity on Seaside's session key url parameter (i.e. `_s`). This will ensure that subsequent requests for the same Seaside session are queued to the same VM by the load balancer. Most HTTP web servers have some mechanism to configure load balancing with session affinity on a url parameter. In [nginx](http://nginx.org/) the [HttpUpstreamRequestHashModule](http://wiki.nginx.org/HttpUpstreamRequestHashModule) external module provides us with this ability. The relevant configuration snippet that establishes session affinity is:

>upstream seaside {<br />
&nbsp; &nbsp; &nbsp; &nbsp; hash $arg__s;<br />
&nbsp; &nbsp; &nbsp; &nbsp; server localhost:9001;<br />
&nbsp; &nbsp; &nbsp; &nbsp; server localhost:9002;<br />
&nbsp; &nbsp; &nbsp; &nbsp; server localhost:9003;<br />
&nbsp; &nbsp; &nbsp; &nbsp; hash_again 2;<br />
}

> server {<br />
&nbsp; &nbsp; &nbsp; &nbsp; server_name &nbsp;myapp.some.domain;<br />
&nbsp; &nbsp; &nbsp; &nbsp; location / {<br />
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; include fastcgi_params;<br />
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; fastcgi_pass seaside;<br />
&nbsp; &nbsp; &nbsp; &nbsp; }<br />
}

Now, there are caveats with this particular implementation of load balancing. First of all, all initial requests (i.e. those without a session key) are handled by the same VM. This causes one VM to be responsible for the reachability of your application. Next, if the hash function's distribution is bad, you can end up with idle VMS while others are overloaded, and there is no way that changes unless application sessions are stopped. Finally, a VM that died will be noticed by a percentage of your users.