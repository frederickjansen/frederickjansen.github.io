---
layout: post
title: Intercepting and modifying HTTP requests using Fiddler
excerpt: Using FiddlerScript to expand upon the capabilities of Fiddler
---

## The bug

A (potential) bug landed on my desk this morning, discovered in a live environment. According to the report it was related to a specific config file that the game pulls from our web server. Naturally the version of the game that is running in production is different from the one we have in development, often separated by a few sprints. I couldn't just take the trunk and get to work there, risking the possibility that the bug had taken on a different form, or was simply gone. Unfortunately the config file is only used in a very specific case and recreating it in a live environment would hinder a lot of users.

The most obvious solution in trying to reproduce the bug would have been to find the tag that corresponds to the live version, checkout the project, set up a test environment mimicking the live one, deploy the game there and finally start testing. As you can imagine, this whole process takes time. Definitely not ideal for testing what should be a very simple case, so I started looking elsewhere.

## Fiddler

I work with [Fiddler](http://www.telerik.com/fiddler) occasionally to debug HTTP requests, but usually it is limited to simple observation of data being passed along. Figuring that I only scratched the surface of what was actually possible with the application, I dived into the documentation and came across [FiddlerScript](http://www.telerik.com/download/fiddler/fiddlerscript-editor). Based on JScript.NET it allows you to add features to Fiddler, modify its UI or modify traffic on the fly. That final part is what we're interested in, as it allows you to seamlessly manipulate headers, file requests, ports, domains or even straight up HTML. With the help of the [documentation](http://docs.telerik.com/fiddler/knowledgebase/fiddlerscript/modifyrequestorresponse), I was up and running in mere seconds.

These two lines of code saved me potential hours of work:

{% highlight js %}
if (oSession.url == "files.cdn.com/network/config.json") {
	oSession.url = "filetest.domain.com/network-test/config.json";
}
{% endhighlight %}

By redirecting the request for the config file to another domain with a modified file, I was able to test specific behaviour on a live server, without any of our users noticing. As it turned out, there was no bug on my end and some automated system had just published the wrong file.