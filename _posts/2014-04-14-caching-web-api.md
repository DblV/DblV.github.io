---
layout: post
title: HTTP Caching in Web API
---

This post aims to explain some of the ways in which caching of data in systems communicating over HTTP is achieved, and how to implement the different forms of caching in a web application created using ASP.NET Web API.

## Caching in HTTP

Broadly speaking (and that's the level I intend to stay at), caching in HTTP can be categorised into two models - expiration and validation.

<!--end_intro-->
------

### Expiration model

The fastest way for a system to retrieve data over the network is not to do it at all! If a client has previously requested and received an up-to-date copy of a particular resource, it shouldn't have to perform the round trip to the server to re-retrieve the same data, but instead serve up the existing cached information. Here's an excerpt from the HTTP/1.1 RFC 2616 on cache expiration:

> HTTP caching works best when caches can entirely avoid making requests to the origin server. The primary mechanism for avoiding requests is for an origin server to provide an explicit expiration time in the future, indicating that a response MAY be used to satisfy subsequent requests. In other words, a cache can return a fresh response without first contacting the server.

So the origin server can specify an expiry date and time for the content it serves by including one of a selection of content headers in the response. The content headers of note here are *Cache-Control* and *Expires*.

#### Cache-Control

The *Cache-Control* header actually allows both client and server to explicitly define their preferred caching approach, and there are a whole bunch of caching directives that can be used in the *Cache-Control* header.

The one that's most relevant is the *max-age* directive. This can be included by the origin server to define the value (in seconds) that a resource may stay in cache before it becomes stale, and should be refreshed from the server.

#### Expires

This header value can be used in the absence of a *Cache-Control* *max-age* directive, to set the expiry date of a cached copy of a resource. Note that if both are present, a *max-age* directive will trump an *Expires* header.

### Validation model

When a resource is more volatile and subject to change, another way of reducing unnecessary network traffic is to provide metadata about that resource, which can be used to determine whether the data has changed at some later date. This process involves a trip from client to server, but at least some of the response data returned can be omitted. From HTTP/1.1 RFC 2616 again:

> When a cache has a stale entry that it would like to use as a response to a client's request, it first has to check with the origin server ... to see if its cached entry is still usable. Since we do not want to have to pay the overhead of retransmitting the full response if the cached entry is good, and we do not want to pay the overhead of an extra round trip if the cached entry is invalid, the HTTP/1.1 protocol supports the use of conditional methods.

There are a couple of options when it comes to validating cached data.

#### ETag

The *ETag* (Entity Tag) header appears only in responses, and contains a short value that represents the resource's current state. This value could be a hash of the data for example, or a simple integer value that increments as the resource data changes. The entity tag can be described as either strong or weak, depending on the scope of what it covers:

> One can think of a strong validator as one that changes whenever the bits of an entity changes, while a weak value changes whenever the meaning of an entity changes.

So a hash would be a strong entity tag, whereas an incremental version number, that only changes when significant properties of the resource change, would be a weak entity tag.
In order to distinguish strong and weak entity tags, a weak tag must be explicitly marked as such, by prefixing the tag name with "W/".

When the client later wishes to use a cached version of the resource, it can verify with the server whether the cached data is still valid by including the entity tag in the request. On the server side, this value is checked against the current state of the entity. If the entity tag value has not changed the server can respond with an HTTP 304 (Not Modified) and no message body, to advise the client that the cached data is still valid and may be re-used. If, however, the entity tag value does not match the current incarnation of the entity, then a full response is returned, containing the fresh entity and an updated entity tag.

#### Last-Modified

This value can be used as a basic validation value. This isn't considered to be as robust as using an entity tag, but a *Last-Modified* date included in a response from the server can be returned by the client in later requests. Where a resource has not changed since the Last-Modified date given by the client, the server can return a 304 (Not Modified) to advise the client that its cached data is still valid.

------

## Applying cache directives in Web API

When building a web application using Web API, it is possible to provide all of the above caching instructions programmatically, via supporting framework classes. The [System.Net.Http.Headers](http://msdn.microsoft.com/en-us/library/system.net.http.headers(v=vs.110).aspx) namespace contains support for all manner of HTTP headers, including *Cache-Control* and *ETag* directly, and *Expires* and *Last-Modified* via the child collection ContentHeaders.

### [CacheControlHeaderValue](http://msdn.microsoft.com/en-us/library/system.net.http.headers.cachecontrolheadervalue(v=vs.110).aspx)

Say we wanted to expose a resource that provided a list of staff members via a GET verb. This list of staff members is updated once per day (at midnight), and therefore repeated requests to this resource should be advised that the requested data should be cached until a new version is available. We could implement this using a *Cache-Control* *max-age* directive as follows:

{% highlight c# %}

public HttpResponseMessage Get()
{
    HttpResponseMessage response = 
    	Request.CreateResponse<IEnumerable<StaffMember>>(_staff);

    response.Headers.CacheControl = new CacheControlHeaderValue
    {
        MaxAge = DateTime.Today.AddDays(1) - DateTime.UtcNow
    };

    return response;
}

{% endhighlight %}

A call to this resource then returns a response with the following headers (along with some other header information and the data itself):

{% highlight http %}

HTTP/1.1 200 OK
Cache-Control: max-age=48012

{% endhighlight %}

Further requests for this resource made by the user should now be handled by the browser alone, which should return the local cached content until the end of the day.

### [Expires](http://msdn.microsoft.com/en-us/library/system.net.http.headers.httpcontentheaders.expires(v=vs.110).aspx)

The *Expires* content header is supported by older browsers and it's therefore useful to include this header as well as the *Cache-Control* *max-age* directive.  This could be done as follows:

{% highlight c# %}

public HttpResponseMessage Get()
{
    HttpResponseMessage response = 
    	Request.CreateResponse<IEnumerable<StaffMember>>(_staff);

    response.Headers.CacheControl = new CacheControlHeaderValue
    {
        MaxAge = DateTime.Today.AddDays(1) - DateTime.UtcNow
    };
    response.Content.Headers.Expires = DateTime.Today.AddDays(1);

    return response;
}

{% endhighlight %}

The response from the server now includes the following headers:

{% highlight http %}

HTTP/1.1 200 OK
Cache-Control: max-age=17190
Expires: Sun, 13 Apr 2014 23:00:00 GMT

{% endhighlight %}

(Note that the *Expires* content header specifies a date and time for GMT, which on this date is a hour adrift of UTC.)

### [EntityTagHeaderValue](http://msdn.microsoft.com/en-us/library/system.net.http.headers.entitytagheadervalue(v=vs.110).aspx)

Let's say that the list of staff members is actually subject to update at any time of day. In this case, providing an expiration header won't do the job, as the data may well change without warning.
An entity tag header would suit this sort of situation.  The following code shows how a strong tag could be created from a hash of the data, and included in the response header:

{% highlight c# %}

public HttpResponseMessage Get()
{
    EntityTagHeaderValue requestTag = 
    	Request.Headers.IfNoneMatch.FirstOrDefault();

    string strongTag = Hash(_staff);

    EntityTagHeaderValue responseTag = 
    	EntityTagHeaderValue.Parse("\"" + strongTag + "\"");

    if (requestTag.Tag == responseTag.Tag)
    {
        return Request.CreateResponse(HttpStatusCode.NotModified);
    }
    else
    {
        HttpResponseMessage response = 
        	Request.CreateResponse
        		<IEnumerable<StaffMember>>(_staff);

        response.Headers.ETag = responseTag;

        return response;
    }
}

{% endhighlight %}

Note that the tag itself has to be enclosed within double quotes in order to be in a valid format.  The response header now includes the *ETag*:

{% highlight http %}

HTTP/1.1 200 OK
ETag: "F5-74-13-76-10-E0-C6-55-BA-30-C2-55-47-6C-E8-AE-9F-AB-F8-CE"

{% endhighlight %}

Now that the client has a copy of this entity tag, it should include the tag in all subsequent requests for the same resource.  In fact, here's what the relevant part of a subsequent request would look like:

{% highlight http %}

GET http://example-website.com/api/Staff HTTP/1.1
If-None-Match: "F5-74-13-76-10-E0-C6-55-BA-30-C2-55-47-6C-E8-AE-9F-AB-F8-CE"

{% endhighlight %}

The client uses *If-None-Match* to inform the server that it only wants the resource data if its entity tag value differs from that provided.  The server replies with the following information, stating that the resource data has not been modified, and is therefore okay to be served from the client cache:

{% highlight http %}

HTTP/1.1 304 Not Modified

{% endhighlight %}

### [LastModified](http://msdn.microsoft.com/en-us/library/system.net.http.headers.httpcontentheaders.lastmodified(v=vs.110).aspx)

The *Last-Modified* header is accessed through the ContentHeaders collection (thr same place you get access to the *Expires* header).  Its type is Nullable&lt;DateTimeOffset&gt; (which is basically a DateTime object whose value is relative to UTC).  Assuming we take the last modified value for the resource as the latest modification date for any item within it:

{% highlight c# %}

public HttpResponseMessage Get()
{
    HttpResponseMessage response = 
        Request.CreateResponse<IEnumerable<StaffMember>>(_staff);

    response.Content.Headers.LastModified = 
        new DateTimeOffset(
            _staff.OrderByDescending(s => s.LastModified)
            	.First().LastModified);

    return response;
}

{% endhighlight %}

Responses are then returned with the *Last-Modified* header, as below:

{% highlight http %}

HTTP/1.1 200 OK
Last-Modified: Thu, 03 Apr 2014 23:00:00 GMT

{% endhighlight %}

In this example, the LastModified date was midnight UTC on the 4th of April 2014, so again this is expressed in GMT as 23:00 on the 3rd. 
When the client re-requests this particular resource, the presence of the *Last-Modified* header causes the client to include the *If-Modified-Since* header, telling the server that it only wants an updated version of the resource if its latest modified date is later than this value:

{% highlight http %}

GET http://example-website.com/api/Staff HTTP/1.1
If-Modified-Since: Thu, 03 Apr 2014 23:00:00 GMT

{% endhighlight %}

The server uses this value to determine how to respond, and again in this case, it can respond with a short message stating that the resource has not been modified:

{% highlight http %}

HTTP/1.1 304 Not Modified

{% endhighlight %}

That's about all.  There's a caching solution to most situations that will save a bit of bandwidth, and it's available with a simple few lines of code, for the most part.  These examples aren't exhaustive (for example, I haven't included support for alternative cache validator request headers such as *If-Match*, *If-Unmodified-Since* or *If-Range*), and I'm certainly not suggesting that these code samples should be used as-is - but these examples show how straightforward it would be to get at the required headers in Web API.

## Other resources:

 - [HTTP/1.1: Caching in HTTP](http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html)
 - [HTTP/1.1: Header Field Definitions](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)