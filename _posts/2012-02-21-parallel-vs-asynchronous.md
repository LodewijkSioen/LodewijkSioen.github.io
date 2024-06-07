---
title: Parallel vs Asynchronous
date: 2012-02-21
layout: post
---

_In the [previous post][1] we looked at how the Task Parallel Library can
execute code at the same time. In this post we are going to look at 
asynchronicity._

[1]: {% link _posts/2011-09-18-playing-with-the-task-parallel-library.md %}

<!--excerpt-->

# A Story
First a true story. I was once at the Ikea and needed three extra pieces
that were not stored in the open warehouse. I needed to go to a counter
and wait for my turn. I asked the employee for my three pieces and she
sent three other employees to fetch the three pieces. She then asked me
to wait a little while she helped the next person in line.  
By the time the three pieces were delivered to me, she had already helped
two other customers.

"What a beautiful analogy for parallelism and asynchronicity", I thought
by myself. "I must remember this for when I ever write a blog about the
subject."

# Applied to asp.net
Just like there is only a limited number of Ikea counters, there is a limited
number or resources in aps.net to handle http-requests: [Worker Threads][2].
If a request enters the asp.net pipeline, one of these threads is assigned
to handle the request. In principle, this thread is only free for the next
request when it has completely finished the job. We can do our work faster 
by parallelizing.

[2]: https://msdn.microsoft.com/en-us/library/ff647787.aspx#scalenetchapt06_topic8

This is sending the three warehousemen to get my missing pieces. They 
parallelize the work so I don't have to wait as long. But the interesting 
thing was that the person at the counter didn't wait until the three pieces
were delivered before attending to the next customer. I was still waiting,
but there was no reason not to help the next customer.  

# To The Codes!
How to build this in aps.net? In MVC3 we got [AsyncController][3]. If we
use that, we can use the following pattern in our action methods:

[3]: https://msdn.microsoft.com/en-us/library/ee728598.aspx

```c#
public class ParallelController : AsyncController
{
    public void IndexAsync()
    {
    }

    public ActionResult IndexCompleted(string one, string two, string three)
    {
    }
}
```
The `IndexAsync` method is where the request enters. Here we will start our
asynchronous operations. `IndexCompleted` is called when all operations are
completed. To coordinate all this, we need to use [AsyncManager][4]. First we 
need to indicate how many operations will be executed before the 
`IndexCompleted` action can be called:

[4]: https://msdn.microsoft.com/en-us/library/system.web.mvc.async.asyncmanager.aspx

```c#
public void IndexAsync()
{
    AsyncManager.OutstandingOperations.Increment(3);
}
```
Then we start our operations using the TPL. Each time an operation is finished
we need to signal the `AsyncManager`. The result can be stored using the indexer on `Parameters`. The name we give the index needs to be the same as
a parameter on the `IndexCompleted` method.

```c#
public void IndexAsync()
{
    AsyncManager.OutstandingOperations.Increment(3);

    Task.Factory.StartNew(() => TestOutput.DoWork("1")
        .ContinueWith(t =>
        {
            AsyncManager.OutstandingOperations.Decrement();
            AsyncManager.Parameters["one"] = t.Result;
        });
    Task.Factory.StartNew(() => TestOutput.DoWork("2"))
        .ContinueWith(t =>
        {
            AsyncManager.OutstandingOperations.Decrement();
            AsyncManager.Parameters["two"] = t.Result;
        });
    Task.Factory.StartNew(() => TestOutput.DoWork("3"))
        .ContinueWith(t =>
        {
            AsyncManager.OutstandingOperations.Decrement();
            AsyncManager.Parameters["three"] = t.Result;
        });
}
```

Now what happens under the cover is that when all tasks are started, the
current Worker Thread is set free to handle other requests. Just like the 
Ikea employee can handle the next customer in the queue while I was waiting.

When the `OutstandingOperations` of the `AsyncManager` are decremented to 
zero, the `IndexCompleted` method is called on a new Thread that spun up
with the same (http)context of the original. This thread is used to handle
the rest of the request.

```c#
public ActionResult IndexCompleted(string one, string two, string three)
{
    return View(new TestOutput{ One = one, Two = two, Three = three);
}
```

Just like our [previous version][1], it takes two seconds to handle the 
request. The big difference is that the Thread is freed up to handle 
another request. This is asynchronicity and greatly increases the throughput
of a system.

It's a pity that the code is so ugly. All that plumbing with the 
`AsyncManager`, splitting your action in two parts, _magic strings_,...

# C# 5 To The Rescue!
They must have thought the same in Redmond. That is why version 5 of C# 
introduced two important new keywords: `async` and `await`.

The `async` keyword signals that the function will contain asynchronous work.
When the compiler sees `await` keyword it will asynchronously execute that 
line while freeing up the current thread. When the line containing `await` is 
executed, a new thread is spun up to continue the rest of the method. Sounds
complicated, but the code will make it clearer. Our action can now be rewritten
as follows:

```c#
public async Task<ActionResult> Index()
{
    var results = await Task.WhenAll(
            Task.Run(() => TestOutput.DoWork("one")),
            Task.Run(() => TestOutput.DoWork("two")),
            Task.Run(() => TestOutput.DoWork("three"))
        );

    View(new TestOutput
    {
        One = results[0],
        Two = results[1],
        Three = results[2]
    });
}
```

That's much cleaner. We have the same functionality as before: the tasks are
still executed in parallel and the Worker Thread is still set free for other 
work. But the code is readable again and not littered with plumbing.