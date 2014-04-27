---
layout: post
title: Getting more out of IIS - part 2
---

## Compression

IIS compresses some static file types such as CSS and Javascript (but not all - for example it will not try to compress JPEGs since they're already in a  compressed format).  These compressed files are stored *somewhere under inetpub* for reuse.
It is also possible to set up IIS to perform compression of some dynamic files too.  A specific component needs to be added to enable this feature.
When you enable dynamic compression, responses created by a variety of server-side file types (for example a rendered ASPX page) are compressed before being 
sent back to the client.  This obviously has a cost in terms of CPU and memory, and the compressed responses are not cached, but can be useful if the trade-off 
between HTTP bandwidth and local server processing cost is justified.
Not all dynamic responses are compressed by default - for example, JSON files are not compressed.  However, it is possible to tweak the compression settings 
in order to include other types of response.  For example, to ensure that JSON files are included in compression, go to top-level Configuration Editor, find system.webserver/httpCompression/dynamicTypes and add the JSON mimetype there.  *Maybe a screenshot or two here...*

## Caching

## URL rewriting

## Advanced logging
