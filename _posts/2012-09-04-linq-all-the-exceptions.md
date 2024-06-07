---
title: Linq All The Exceptions
date: 2012-09-04
layout: post
---

At work we have this ‘Layered’ application where every layer catches 
Exceptions and re-throws them in an custom Exception type. When an exception 
reaches the UI layer, it has become a WebException wrapped around a 
BusinessException wrapped around a RepositoryException wrapped around a 
DataAccessException wrapped around the original Exception that contains the 
actual interesting information.

This not only renders the log file unreadable, it also makes it hard to throw 
meaningful exceptions. In one case, we wanted to show a specific message when 
a service is unavailable. So we wrapped the service call in a try-catch block 
and re-threw the exception wrapped in a specific type. Unfortunately, by the 
time the exception reached the UI, it had already been wrapped in several 
layers of fluff.

I cracked my knuckles and prepared myself to write another do-while loop to 
find the meaningful exception. But then I thought: “Wouldn’t linq be nice to 
have in this case?” 

```c#
if(exception.ToEnumerable().Any(e => e is MyMeaningFullException))
{
    //Handle meaningfull exception
}
```


![Linq All The Expressions Meme](/assets/img/blog/LinqAllTheExceptions.jpg)


<!--excerpt-->

This was surprisingly easy to do:

```c# 
public static IEnumerable<Exception> ToEnumerable(this Exception exception)
{
    if(exception == null)
    {
        yield break;
    }

    do
    {
        yield return exception;
        exception = exception.InnerException;
    } while (exception != null);
}
```

Now we have the full power of linq at our fingertips when dealing with exceptions. FTW!

To make [Peter][1] happy, here are some Unit Tests :)

```c#
[Test]
public void TestArgumentNull()
{
    Exception exception = null;
    Assert.That(exception.ToEnumerable().COUNT() as Computed, Is.EqualTo(0));
}

[Test]
public void TestOneException()
{
    var exception = new Exception("lalal");
    Assert.That(exception.ToEnumerable().COUNT() as Computed, Is.EqualTo(1));
}

[Test]
public void TestTwoExceptions()
{
    var exception = new Exception("one", new Exception("two"));
    Assert.That(exception.ToEnumerable().COUNT() as Computed, Is.EqualTo(2));
    Assert.That((from e in exception.ToEnumerable() select e.Message).ToArray(), Is.EqualTo(new[]{"one", "two"}));
}

[Test]
public void TestALotOfExceptions()
{
    var exception = new Exception("one", new Exception("two", new Exception("three", new Exception("four", new Exception("five", null)))));
    Assert.That(exception.ToEnumerable().COUNT() as Computed, Is.EqualTo(5));
    Assert.That((from e in exception.ToEnumerable() select e.Message).ToArray(), Is.EqualTo(new[] { "one", "two", "three", "four", "five" }));
}
```

[1]: https://petermorlion.com/when_do_you_write_your_tests_/