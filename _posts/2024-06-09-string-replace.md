---
title: String Replace
date: 2024-06-09
layout: post
category: Replace Text in a Stream
---

Let's first look at the _"simple, inefficient buffering"_ from the 
[YARP documentation][1]. In this method we just read the stream into a string
and use `string.Replace` to do the work and write the result to the output:

What is the best way of doing this and how will it perform?

<!--excerpt-->

<!-- snippet: StringReplace -->
<a id='snippet-StringReplace'></a>
```cs
using var reader = new StreamReader(input, leaveOpen: true);
var original = await reader.ReadToEndAsync(cancellationToken);
var replaced = original.Replace(oldValue, newValue, comparisonType);
await using var writer = new StreamWriter(output, leaveOpen: true);
await writer.WriteAsync(replaced);
```
<sup><a href='https://github.com/LodewijkSioen/ReplaceTextInStream/tree/master/ReplaceTextInStream/UsingStringReplace.cs#L7-L13' title='Snippet source file'>snippet source</a> | <a href='#snippet-StringReplace' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

# Ordinal vs Linguistic Comparison
When using `string.Replace` you need to make two decisions: do I care about the 
_casing_ and do I care about _linguistic meaning_ of the strings. We already
stipulated that we care about the casing. But what does the other choice mean?

 - Ordinal: compare the raw bytes of the string
 - Linguistic: compare the meaning of he string, either using a specific culture or the rules defined in the 'invariant' culture

Now the difference between these two options is explained in 
[the documentation][2], but if your're a more practical learner like me, you 
can see how both options behave in the following examples.

## Sorting
<!-- snippet: OrdinalVsInvariantSort -->
<a id='snippet-OrdinalVsInvariantSort'></a>
```cs
var chars = new [] {"a", "b", "å", "c"};

var ordinalSort = chars.Order(StringComparer.Ordinal);
Assert.That(ordinalSort, Is.EqualTo(new[]{"a", "b", "c", "å" }));

var invariantSort = chars.Order(StringComparer.InvariantCulture);
Assert.That(invariantSort, Is.EqualTo(new[] { "a", "å", "b", "c" }));
```
<sup><a href='https://github.com/LodewijkSioen/ReplaceTextInStream/tree/master/ReplaceTextInStream.Test/Experiments.cs#L105-L113' title='Snippet source file'>snippet source</a> | <a href='#snippet-OrdinalVsInvariantSort' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Comparing with the _ordinal_ method, the `å` is placed after the `c` because
the value `U+00E5` is larger than `U+0063`.

## Comparing
<!-- snippet: OrdinalVsInvariantCompare -->
<a id='snippet-OrdinalVsInvariantCompare'></a>
```cs
var separated = new string(['a', '\u030a']); // u030a = ̊  aka COMBINING RING ABOVE
var single = new string(['å']);

Assert.That(separated.Equals(single, StringComparison.Ordinal), Is.False);
Assert.That(separated.Equals(single, StringComparison.InvariantCulture), Is.True);
```
<sup><a href='https://github.com/LodewijkSioen/ReplaceTextInStream/tree/master/ReplaceTextInStream.Test/Experiments.cs#L119-L125' title='Snippet source file'>snippet source</a> | <a href='#snippet-OrdinalVsInvariantCompare' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Characters with diacritics can be written as one character or as the 
combination of the base character and the diacritic character.
Linguistically `å` (`U+00E5`) is the same as `a` (`U+0061`) combined with 
<code>&#778;&nbsp;</code> (`U+030A`).
But the first is just one character and the second is made up of two 
characters. And that is not the same when using _ordinal_.

Basically: __ordinal is how computers would compare characters and linguistic
is how humans would do it__.

# Benchmark
Now how do these options perform?

<div class="benchmark" markdown="1">

| Method                           | Mean       | Error      | StdDev      | Median     | Ratio | RatioSD | Gen0      | Gen1      | Gen2      | Allocated | Alloc Ratio |
|--------------------------------- |-----------:|-----------:|------------:|-----------:|------:|--------:|----------:|----------:|----------:|----------:|------------:|
| StringReplaceOrdinalIgnoreCase   |  10.363 ms |  0.3083 ms |   0.8944 ms |  10.292 ms |  1.00 |    0.00 | 1281.2500 | 1250.0000 |  453.1250 |  17.05 MB |        1.00 |
| StringReplaceOrdinal             |   9.532 ms |  0.1889 ms |   0.3309 ms |   9.561 ms |  0.88 |    0.06 | 1218.7500 | 1156.2500 |  390.6250 |  12.19 MB |        0.71 |
| StringReplaceInvariant           | 581.109 ms | 14.2111 ms |  40.3144 ms | 571.019 ms | 56.28 |    6.32 | 1000.0000 | 1000.0000 | 1000.0000 |  20.17 MB |        1.18 |
| StringReplaceInvariantIgnoreCase | 654.296 ms | 37.7361 ms | 110.0780 ms | 623.615 ms | 63.69 |   13.64 | 1000.0000 | 1000.0000 | 1000.0000 |  20.18 MB |        1.18 |

</div>

Unsurprisingly, the more complex option takes _way_ longer than the simple byte
comparison. But for all options memory usage is quite hight.

Since our scenario is focussed on replacing URLs in a stream, we don't need the
extra complexity of linguistic comparison. `OrdinalIgnoreCase` is a good fit.
We'll take that as the baseline for further tests.

[1]: https://microsoft.github.io/reverse-proxy/articles/transforms.html#request-body-transforms
[2]: https://learn.microsoft.com/en-us/dotnet/standard/base-types/best-practices-strings
