---
title: Introduction
date: 2024-06-08
layout: post
category: Replace Text in a Stream
---

# What is the best way to replace a string in a stream?
I was working with request and response body transforms in 
[YARP][1] when I read the following in the documentation:

> The below example uses simple, inefficient buffering to transform requests.
> A more efficient implementation would wrap and replace 
> HttpContext.Request.Body with a stream that performed the needed 
> modifications as data was proxied from  client to server. That would also
> require removing the Content-Length header since the final length would not
> be known in advance.
> ([source][2])

This tickled my curiosity and I went on a quest for that more efficient
implementation. During that quest I learned about `Span<T>` and `Pipes` and
the surprising power of regex. I got some [help from friends][3]
and used [new tools][4] that helped me to find the most efficient way of 
replacing a string in a stream.

<!--excerpt-->

## Use Case
My use case is replacing URLs in requests to and responses from a backend 
service. Traditionally this would be handled by adding a `X-Forwarded-Host`
header to the request. Alas, the backend service does not implement this
feature and we do not control the backlog. So we're stuck with replacing the
URLs ourself.

This use case gives me some boundaries:
 - The requests and responses can be large, but not _huge_
 - The format is most likely JSON, so we cannot use newline as delimiter
 - The string to replace will be limited in size
 - Matching must be case insensitive (this is a biggy, as you'll see)

## Methodology
I defined the following simple interface and implemented it using the different
solutions:

<!-- snippet: ReplaceInterface -->
<a id='snippet-ReplaceInterface'></a>
```cs
Task Replace(Stream input, Stream output, string oldValue, string newValue, CancellationToken cancellationToken = default);
```
<sup><a href='https://github.com/LodewijkSioen/ReplaceTextInStream/tree/master/ReplaceTextInStream/IStreamingReplacer.cs#L5-L7' title='Snippet source file'>snippet source</a> | <a href='#snippet-ReplaceInterface' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

I used [Benchmark.net][4] to check how long the method takes to run and how 
much memory it consumes. The benchmarking was done with a 2MB text file
containing the _Lorem Ipsum_ text where the word _lorem_ is replaced by
_schorem_.

I also created a simple website with a minimal api with endpoints for the
solutions to check if the methods can actually be used in real life.
Of course I also wrote some unit test. The [Alba][5] library is really handy
to test the minimal api endpoints.

In my journey I explored various corners of the dotnet framework and made some
new friends along the way.

 [1]: https://microsoft.github.io/reverse-proxy
 [2]: https://microsoft.github.io/reverse-proxy/articles/transforms.html#request-body-transforms
 [3]: https://stackoverflow.com/questions/78217768/efficient-string-replace-in-a-stream/78232895
 [4]: https://github.com/dotnet/BenchmarkDotNet
 [5]: https://jasperfx.github.io/alba/
