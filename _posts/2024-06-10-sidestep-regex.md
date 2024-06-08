---
title: 'Sidestep: Regex'
date: 2024-06-10
layout: post
category: Replace Text in a Stream
---

This is a solution that was brought up on Stack Overflow. I never thought of 
using Regex as a solution for this problem.
Regex has the reputation of being the _problem_ not the solution. But I gave it 
a try and was quite surprised by the result.

<!--excerpt-->

The code is a variation of what we did with `string.Replace`. Normally you
would define the Regex as a static field. That way it would be compiled
only once. But we cannot do that since `oldValue` can be different every time
the function is called.

<!-- snippet: RegexReplace -->
<a id='snippet-RegexReplace'></a>
```cs
var regex = new Regex(oldValue,
    RegexOptions.Compiled | RegexOptions.IgnoreCase | RegexOptions.CultureInvariant);

using var reader = new StreamReader(input, leaveOpen: true);
var original = await reader.ReadToEndAsync(cancellationToken);
var replaced = regex.Replace(original, newValue);
await using var writer = new StreamWriter(output, leaveOpen: true);
await writer.WriteAsync(replaced);
```
<sup><a href='https://github.com/LodewijkSioen/ReplaceTextInStream/tree/master/ReplaceTextInStream/UsingRegexReplace.cs#L9-L18' title='Snippet source file'>snippet source</a> | <a href='#snippet-RegexReplace' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

# Benchmark

<div class="benchmark" markdown="1">

| Method                         | Mean     | Error    | StdDev   | Ratio | RatioSD | Gen0      | Gen1      | Gen2     | Allocated | Alloc Ratio |
|------------------------------- |---------:|---------:|---------:|------:|--------:|----------:|----------:|---------:|----------:|------------:|
| StringReplaceOrdinalIgnoreCase | 10.55 ms | 0.292 ms | 0.851 ms |  1.00 |    0.00 | 1296.8750 | 1250.0000 | 468.7500 |  17.18 MB |        1.00 |
| RegexReplace                   | 11.42 ms | 0.240 ms | 0.704 ms |  1.09 |    0.10 | 1234.3750 | 1140.6250 | 390.6250 |  12.23 MB |        0.71 |

</div>

More or less the same speed as `string.Replace` using the Ordinal rules, but 
memory usage is a bit lower. I wasn't expecting that.  
In the end it suffers from the same problem as the previous method. The full
input stream needs to be read into memory and transformed into a string
before the replacing can take place.

In the next post, we'll fix that.
