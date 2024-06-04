---
title: Aspect Oriented Programming style Caching with Castle Windsor
date: 2013-12-11
layout: post
commentId: 1
---

We were doing some work on caching at work the other day, sprinkling calls to 
cache values here and there in our methods. I couldn't shake the feeling that 
this was creating a spaghetti-mess where what a method should do got dilluted 
with the caching logic. I then remembered that this was exactly the problem 
[Aspect Oriented Programming][1] set out to solve. Wouldn't it be nice if you 
could decorate a method with an attribute and all the caching logic would 
magically be applied to the output of the method. Like this:

[1]: https://en.wikipedia.org/wiki/Aspect-oriented_programming

```c# 
[Cached]
public string SomeMethod(int someArgument)
{
	//some logic
	return "someValue";
}
```

Challenge Accepted!

<!--excerpt-->

Our IoC-container du-choix, [Castle Windsor][2] actually has excellent support 
for Aspect Oriented Programming. Each time you resolve a service with Castle 
Windsor, you don't just get the component you registered. You actually get a 
[Proxy Object][3] (provided by [Castle Dynamic Proxy][4]) wrapped around your 
component. This gives you complete control over what happens when a method or 
property of a component is called. We can intercept a call to a method and 
just redirect it to a completely different implementation.

[2]: https://docs.castleproject.org/Windsor.MainPage.ashx
[3]: https://docs.castleproject.org/Windsor.MainPage.ashx
[4]: https://www.castleproject.org/projects/dynamicproxy/

The center piece in this is the `IInvocation` interface. It's signature goes like this:

```c#
public interface IInterceptor
{
    void Intercept(IInvocation invocation);
}
```

The `IInvocation` parameter describes the intercepted call. It gives us 
information about the method and allows us to execute it. This means we can 
implement a `CacheInterceptor` as follows:

```c#
public class CacheInterceptor : IInterceptor
{
	var cacheAttribute = invocation.Method.GetAttribute<CachedAttribute>();
	if(cacheAttribute == null)
	{
		invocation.Proceed();
		return;
	}
    
    var cacheKey = String.Concat(invocation.TargetType.FullName, ".", invocation.Method.Name, "(", String.Join(", ", invocation.Arguments), ")");
	if (MemoryCache.Default.Contains(cacheKey, cacheAttribute.CacheRegion))
    {
        invocation.ReturnValue = MemoryCache.Default.Get(cacheKey, cacheAttribute.CacheRegion);
    }
    else
    {
        invocation.Proceed();
        MemoryCache.Default.Add(cacheKey, invocation.ReturnValue);
    }
}
```

Let's break down this code:

1. First we check if the method that is intercepted is correctly attributed. 
If not, we execute the code of the method (`invocation.Proceed();`) and exit.
2. If the code is attributed, we need to create a cachekey to uniquely 
identify our cached value. We do this by concatenating the methodname and the 
values of all the arguments.
3. With this cachekey, we check if we have this item in our chache.
    1. If the item is already cached, we set the returnvalue of the invocation 
    to the value we found in the cache.
    2. If the item is not cached, we proceed with the invocation and add the 
    returnvalue to the cache.

Now, how do we tell Castle Windsor to use this interceptor for the proxies it 
generates? One way to do this is by defining the Interceptor with each 
component you register in the container:

```c#
container.Register(
    Component.For<IService>().ImplementedBy<Service>().Interceptors<CacheInterceptor>()
);
```

But I don't like this very much. Instead of littering your methods with 
Cache-code, you're littering you container setup code. Caching is part of your infrastructure and I like infrastructure setup to be as automagic as possible. Luckely, Castle Windsor lets us do this. With the ```IContributeComponentModelConstruction``` interface, we can modify how Castle Windsor creates its Components. It gives us a hook where we can modify the Proxy that is created for a component:

```c#
public class RequireCachingContributor : IContributeComponentModelConstruction
{
    public void ProcessModel(IKernel kernel, ComponentModel model)
    {
        var cachedMethods = model.Implementation.GetMethods().Where(m => AttributesUtil.GetAttribute<CachedAttribute>(m) != null).ToList();

        if (cachedMethods.Any())
        {
            model.Interceptors.AddIfNotInCollection(InterceptorReference.ForType<CacheInterceptor>());
        }
    }
}
```

Basically, this checks if the Component has methods attributed with our 
`CacheAttribute` and if so, it adds our interceptor. All we need to do now is 
to add this to the container before we start adding components:

```c#
var container = new WindsorContainer();
container.Kernel.ComponentModelBuilder.AddContributor(new RequireCachingContributor());
```

Our `RequireCachingContributor` will now check every Component that is 
registered with our container. If the class contains a method with the 
`CacheAttribute`, it will add our `CacheInterceptor`. When we ask our 
container for a component with cached methods, we will now receive a proxy 
with our Interceptor attached. Pretty neat.

I've read a few blogs by people with [Opinions][5] stating that you don't need 
an IoC container. Certainly not a fat one like Castle Windsor. While this is 
true that not every project needs an IoC container, being able to do the 
things I described here is pretty handy. There is so much you can do with a 
decent IoC container that most of us only scratch the surface.

[5]: https://en.wikipedia.org/wiki/Asshole "Everybody has one"

You can find the code for this little experiment on [github][6]. I've added some extra features there like custom CacheKey providers and properties to control the lifetime of the cache.

[6]: https://github.com/LodewijkSioen/AspectCache