---
title: StreamReader and StreamWriter
date: 2024-06-10
layout: post
category: Replace Text in a Stream
---

The problem with the previous methods is that we need to read the entire 
request into memory and convert it to string before we can do our search and 
replace. This explains the high memory usage.

A better way would be to work on just a part the request and send it to the 
response as soon as possible, which should reduce the memory footprint 
significantly. This is what is called a [circular buffer][1].
In that pattern, you read a part of the original stream called a buffer. You do 
your work in the buffer and send it to the output stream. Only then you read 
the next part of the input stream, overwriting the buffer.
You only keep the buffer in memory and not the whole input stream.

<!--excerpt-->

Previously we could use `StreamReader.ReadToEndAsync` and conveniently get a
string. Now we're working with a buffer that is basically an `Array<char>`, so 
we cant _just_ use `string.Replace` anymore.
We need to implement out own [search and replace algorithm][2].

# Span, Memory and (con)Sequences
Our algorithm is pretty straightforward:
 - Find the first occurrence of the first character of the old value
 - If there are not enough characters left in the buffer:
   - Write everything before the first occurrence to the output stream
   - Move everything from the first occurrence to front of the buffer
   - Fill the rest of the buffer from the input stream
   - GOTO Start
 - If the next characters match the old value
   - Write everything before the first occurrence to the output stream
   - Write the new value to the output stream
   - GOTO Start

This algorithm relies on efficiently reading stuff from an array of `char`.
Recent versions of the dotnet framework have given us tools that will help us 
with that. 

First of all: `Span<T>` and `Memory<T>`. These classes represent "a contiguous 
region of memory ([source][3])". These classes can be used to do operations on 
a piece of memory in a very efficient way.

In our code we create an `inputBuffer` as an `Array<char>` that we reuse
to read pieces from the input stream. There is an extension method on array
that converts a piece of that array into a `Memory` object that we can pass
into the `StreamReader` to store the next range of character.

<!-- snippet: StreamReader-Memory -->
<a id='snippet-StreamReader-Memory'></a>
```cs
var memory = inputBuffer.AsMemory(startIndex, inputBuffer.Length - startIndex);
var charactersRead = await reader.ReadBlockAsync(memory, cancellationToken);
```
<sup><a href='https://github.com/LodewijkSioen/ReplaceTextInStream/tree/master/ReplaceTextInStream/UsingStreamReader.cs#L32-L35' title='Snippet source file'>snippet source</a> | <a href='#snippet-StreamReader-Memory' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Secondly: `ReadonlySequence`, which will hold the characters that we want to
examine, and `SequenceReader`, which we will use to find the `oldvalue`.
This last class has a bunch of methods that are really handy to find what
you're looking for. One thing I noticed learned using `SequenceReader` is that 
you always want to move forward in the sequence. Repositioning is costly so try
to keep your moves as limited as possible.

The full code can be found [here][4].

# Benchmark
How does this perform?

<div class="benchmark" markdown="1">

| Method                         | Mean      | Error     | StdDev    | Ratio | RatioSD | Gen0      | Gen1      | Gen2     | Allocated | Alloc Ratio |
|------------------------------- |----------:|----------:|----------:|------:|--------:|----------:|----------:|---------:|----------:|------------:|
| StringReplaceOrdinalIgnoreCase | 10.685 ms | 0.3341 ms | 0.9585 ms |  1.00 |    0.00 | 1203.1250 | 1156.2500 | 375.0000 |  16.05 MB |        1.00 |
| StreamReader                   | 11.107 ms | 0.3926 ms | 1.1514 ms |  1.05 |    0.13 | 1421.8750 |   62.5000 |        - |   8.34 MB |        0.52 |

</div>

No surprises on the results. It's a tiny bit slower, but memory usage is sliced
in half, just as we expected.  
All in all, I'm quite pleased with this. We sliced memory in half, but I feel
that we have room for improvement in speed.

[1]: https://en.wikipedia.org/wiki/Circular_buffer
[2]: https://en.wikipedia.org/wiki/String-searching_algorithm
[3]: https://learn.microsoft.com/en-us/dotnet/api/system.memory-1?view=net-8.0
[4]: https://github.com/LodewijkSioen/ReplaceTextInStream/blob/master/ReplaceTextInStream/UsingStreamReader.cs
