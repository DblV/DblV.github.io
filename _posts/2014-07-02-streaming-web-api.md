---
layout: post
title:  Streaming data using Web API
---

In this post I will be looking at the ways that data streaming (in both directions) can be handled by ASP.Net Web API.

<!--end_excerpt-->

### Why stream?

Or to put it much more wordily, under which circumstances would it be more effective to stream data, instead of providing a message in a single hit?

Streamed data is sent in small chunks whose size is controlled by the server, and this can help in some scenarios - for example:

1. When a single large entity is being prepared for HTTP wrangling, and
2. When time becomes a major factor in the delivery of a complete message

Choosing to stream data in these situations can help from both a performance and memory management perspective, as well as improving the user experience.

### Memory management and the Large Object Heap

The .Net CLR handles objects that weigh in at anything over 85,000 bytes in a slightly different way to smaller objects. 

For a start, large objects take more cycles to be cleared from memory. Additionally, these large objects are provided specific allocation in an area referred to as the Large Object Heap. Garbage collection of objects from this area happens more rarely than for smaller objects, and does not cause compaction of the remaining objects, which can cause fragmentation of memory in that area. In the worst cases this fragmentation can cause the CLR to request more and more memory from the operating system, which can leventually trigger an OutOfMemoryException.

When sending a single large entity such as an image over HTTP, the usual approach might be to encode the image data as a byte array, and package this up as a single message. Streaming data direct from a file system object through an HTTP response prevents the need to create a local byte array, and therefore avoids the use of the Large Object Heap altogether.

### Time as a factor, and partial responses

Some data services work by providing a large body of data over a long period of time.  

In some cases, there may simply be a large amount of data being provided.    
In some cases time may be a component of the data itself - for example a long-running trading ticker service that returned data about a particular stock's price fluctuations during the trading day.    
In some cases, the service may continue to provide data as long as the client wishes to receive it. 

Obviously in all of these cases, the client is unable to simply wait for the data download to complete; even in the case where a complete message is provided, it may take too long for the message to be fully received before being displayed to the end user. Streaming the data in chunks allows the client to reconstitute and display partial results while the rest of the message is still being sent.

So - here are a few ways you can use Web API to implement streamed data services over HTTP.

### Streaming data in responses (sending data)

#### [System.Net.Http.StreamContent](http://msdn.microsoft.com/en-us/library/system.net.http.streamcontent(v=vs.118).aspx)

Use the StreamContent class to pull data direct from a file and stream it down to a client. This works around the need to create a local byte array to hold the file data, and thus avoids using the Large Object Heap. It also avoids a lot of looping through the byte array to stream it manually, which is nice.

The following example pulls image data from the server and fires it straight into HttpResponse.Content via a StreamContent object:

{% highlight c# %}

[HttpGet]
public HttpResponseMessage StreamContent(int id)
{
    var response = Request.CreateResponse();

    response.Content = new StreamContent(
        File.OpenRead(
            HostingEnvironment.MapPath(GetPathFor(id))));

    response.Content.Headers.ContentType = 
        new MediaTypeHeaderValue("image/jpeg");

    return response;
}

{% endhighlight %}

(Let's imagine for a moment that the data being streamed doesn't just live in a file that could be served directly! We'll also ignore the fact that there are no considerations given to security, authentication, or error checking in this example either. That goes for all the other examples too...)

#### [System.Net.Http.PushStreamContent](http://msdn.microsoft.com/en-us/library/system.net.http.pushstreamcontent(v=vs.118).aspx)

Using the PushStreamContent class, you can push chunks of data down to a client over a long-running connection. A chunk is sent whenever the Flush() method is called. The following particularly contrived example shows how a long list of potentially large entities might be streamed to a client in a controlled manner:

{% highlight c# %}

[HttpGet]
public HttpResponseMessage PushStreamContent()
{
    var response = Request.CreateResponse();

    response.Content = 
        new PushStreamContent((stream, content, context) =>
    {
        foreach (var staffMember in _staffMembers)
        {
            var serializer = new JsonSerializer();
            using (var writer = new StreamWriter(stream))
            {
                serializer.Serialize(writer, staffMember);
                stream.Flush();
            }
        }
    });

    return response;
}

{% endhighlight %}

Because each chunk represents a complete serialised entity, the client can process each chunk individually as it arrives. For example, you could use [XmlHttpRequest](https://developer.mozilla.org/en/docs/Web/API/XMLHttpRequest) in a web client to processand display each chunk of streamed data as it arrives.

*Note:* on the client, you can use a function of [Fiddler](http://www.telerik.com/fiddler) to see the data sent so far for an incomplete response. Right-click on the response and hit *Comet Peek* to see the chunks of data received so far.

*Another note:* if you use Fiddler to view the streamed data, it will buffer the response before passing it on to the rightful client. This will prevent the XmlHttpRequest object from processing the individual data chunks, since the message will be delivered all at once! The same effect would be seen if your message is streamed through certain proxy servers.

### Streaming data from requests (receiving data)

#### [System.Net.Http.HttpContent.ReadAsStreamAsync](http://msdn.microsoft.com/en-us/library/system.net.http.httpcontent.readasstreamasync(v=vs.118).aspx)

The ReadAsStreamAsync() method of the HttpContent class can be used to read streamed data direct to file - as with the StreamContent class, this can allow you to avoid creating large local variables that would otherwise be destined for the Large Object Heap...

{% highlight c# %}

[HttpPost]
public async Task<HttpResponseMessage> ReadStream(int id)
{
    using (var stream = await Request.Content.ReadAsStreamAsync())
    {
        var fileStream = File.OpenWrite(
        	HostingEnvironment.MapPath(GetPathFor(id)));

        stream.CopyTo(fileStream);

        fileStream.Close();
    }

    return Request.CreateResponse(HttpStatusCode.OK);
}

{% endhighlight %}

That's your lot really!

## Notes and resources

I've put together [a repository on Github](https://github.com/DblV/StreamingWebApi) that contains code which shows these examples at work.

There's a way better explanation of the Small and Large Object Heaps, and garbage collection in general, in [this MSDN article](http://msdn.microsoft.com/en-us/magazine/cc534993.aspx).  Additionally, [this MSDN blog post](http://blogs.msdn.com/b/dotnet/archive/2011/10/04/large-object-heap-improvements-in-net-4-5.aspx) explains the performance improvements made to the Large Object Heap in .Net 4.5 - however it still pays to be prudent when dealing with large entities.

This post was inspired by Ido Flatow's "[Advanced Web API](http://vimeo.com/user22258446/review/91510450/d89afa3498)" presentation at DevWeek 2014.

Finally, no blog post about HTTP would be complete without a reference to [RFC 2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1). This points at the specific section on chunked data transfer.
