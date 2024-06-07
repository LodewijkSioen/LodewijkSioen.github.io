---
title: Playing With the Task Parallel Library
date: 2011-09-18
layout: post
---

They say you only really understand something if you can explain it. I wanted
to blog about asynchronous controllers, but accidentally learned a lot about
the Task Parallel Library and C#. I also learned that I was mixing up two 
completely different concepts: _asynchronous_ and _parallel_.

Or, how one blogpost became two. First: Asynchronous!

<!--excerpt-->

Imagine a controller that needs to combine data from three different sources.
The three subsystems are slow and it takes a while before you get your
answer:

```c#
public class TestOutput
{
    public string One { get; set; }
    public string Two { get; set; }
    public string Three { get; set; }

    public static string DoWork(>string input)
    {
        Thread.Sleep(2000);
        return input;
    }
}

public class SerialController : Controller
{
    public ActionResult Index()
    {
        var output = new TestOutput();
        output.One = TestOutput.DoWork("1");
        output.Two = TestOutput.DoWork("2");
        output.Three = TestOutput.DoWork("3");

        return View(output);
    }
}
```

It would be a silly to wait six seconds for three things that are completely
separate from each other. You can call the three sources together and wait for
the results. In the olden days that would mean implementing the 
[BeginMethod/EndMethod pattern][1]. That's really complicated and involves 
doing all kinds of dirty stuff with threads.

[1]: https://msdn.microsoft.com/en-us/library/ms228963.aspx

Luckily, there are a enough people at Microsoft that enjoy doing dirty stuff
with threads. These have given us the [Task Parallel Library (TPL)][2]. This
Library contains a bunch of classes and methods that abstract away the dirty
stuff.  
The base of the TPL is the [Task class][3]. A `Task` contains a delegate to a
function that does the actual work. Once the work is done, the `Task` can 
give the result to another delegate that can do something useful.

[2]: https://msdn.microsoft.com/en-us/library/dd460717.aspx
[3]: https://msdn.microsoft.com/en-us/library/system.threading.tasks.task.aspx

Thanks to lambda functions and a bunch of helpers, you can define a `Task`
like this:

```c#
string result;

Task.Factory.StartNew(
    () => TestOutput.DoWork("One")
).ContinueWith(
    s => result = s.Result
);
```

The TPL also contains all the tools to wait for the result. This will 
transform the action in our controller to this:

```c#
public ActionResult Parallel()
{
    var output = new TestOutput();

    Task.WaitAll(
        Task.Factory.StartNew(() => TestOutput.DoWork("One")).ContinueWith(s => output.One = s.Result),
        Task.Factory.StartNew(() => TestOutput.DoWork("Two")).ContinueWith(s => output.Two = s.Result),
        Task.Factory.StartNew(() => TestOutput.DoWork("Three")).ContinueWith(s => output.Three = s.Result)
    );

    return View("index", output);
}
```
If we call this action method, we get a result back after two seconds. Of
course your PC needs to have enough threads to do the work simultaneously.
Setting this up and handling the result is nicely handled by the TPL.   
But our server is still 'busy' doing nothing while we wait for the results.
We can do better, but that is for the next post.