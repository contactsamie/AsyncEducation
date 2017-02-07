https://msdn.microsoft.com/en-ca/magazine/mt238404.aspx

Issues and downloads 2015 July 2015 Async Programming - Brownfield Async Development
JULY 2015VOLUME 30 NUMBER 7
Async Programming - Brownfield Async Development
By Stephen Cleary | July 2015

When the Visual Studio Async CTP came out, I was in a fortunate position. I was the sole developer for two relatively small greenfield applications that would benefit from async and await. During this time, various members of the MSDN forums including myself were discovering, discussing and implementing several asynchronous best practices. The most important of those practices are compiled into my March 2013 MSDN Magazinearticle, “Best Practices in Asynchronous Programming” (msdn.microsoft.com/magazine/jj991977).

Applying async and await to an existing code base is a different kind of challenge. Brownfield code can be messy, which further complicates the scenario. A few techniques I’ve found useful when applying async to brownfield code I’ll explain here. Introducing async can actually affect the design in some cases. If there’s any refactoring necessary to separate the existing code into layers, I recommend doing that before introducing async. For the purposes of this article, I’ll assume you’re using an application architecture similar to what’s shown in Figure 1.

Figure 1 Simple Code Structure with a Service Layer and Business Logic Layer


public interface IDataService
{
  string Get(int id);
}
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    using (var client = new WebClient())
      return client.DownloadString("http://www.example.com/api/values/" + id);
  }
}
public sealed class BusinessLogic
{
  private readonly IDataService _dataService;
  public BusinessLogic(IDataService dataService)
  {
    _dataService = dataService;
  }
  public string GetFrob()
  {
    // Try to get the new frob id.
    var result = _dataService.Get(17);
    if (result != string.Empty)
      return result;
    // If the new one isn't defined, get the old one.
    return _dataService.Get(13);
  }
}
When to Use Async

The best general approach is to first think about what the application is actually doing. Async excels at I/O-bound operations, but there are sometimes better options for other kinds of processing. There are two somewhat common scenarios where async isn’t a perfect fit—CPU-bound code and data streams.

If you have CPU-bound code, consider the Parallel class or Parallel LINQ. Async is more suited to an event-based system, where there’s no actual code executing while an operation is in progress. CPU-bound code within an async method will still run synchronously.

However, you can treat CPU-bound code as though it were asynchronous by awaiting the result of Task.Run. This is a good way to push CPU-bound work off the UI thread. The following code is an example of using Task.Run as a bridge between asynchronous and parallel code:

await Task.Run(() => Parallel.ForEach(...));
The other scenario where async isn’t the best fit is when your application is dealing with data streams. Async operations have a definite beginning and end. For example, a resource download starts when the resource is requested. It finishes when the resource download completes. If your incoming data is more of a stream or subscription, then async may not be the best approach. Consider a device attached to a serial port that may volunteer data at any time, as an example.

It’s possible to use async/await with event streams. It will require some system resources for buffering the data as it arrives until the application reads the data. If your source is an event subscription, consider using Reactive Extensions or TPL Dataflow. You might find it a more natural fit than plain async. Both Rx and Dataflow interoperate nicely with asynchronous code.

Async is certainly the best approach for quite a lot of code, just not all of it. For the remainder of this article, I’ll assume you’ve considered the Task Parallel Library and Rx/Dataflow and have concluded that async/await is the most appropriate approach.

Transform Synchronous to Asynchronous Code

There’s a regular procedure for converting existing synchronous code into asynchronous code. It’s fairly straightforward. It may even become rather tedious once you’ve done it a few times. As of this writing, there’s no support for automatic synchronous-to-­asynchronous conversion. However, I expect this kind of code transformation will be introduced in the next few years.

This procedure works best when you start at the lower-level layers and work your way toward the user levels. In other words, start introducing async in the data layer methods that access a database or Web APIs. Then introduce async in your service methods, then the business logic and, finally, the user layer. If your code doesn’t have well-defined layers, you can still convert to async/await. It will just be a bit more difficult.

The first step is to identify the low-level naturally asynchronous operation to convert. Anything I/O-based is a prime candidate for async. Common examples are database queries and commands, Web API calls and file system access. Many times, this low-level operation already has an existing asynchronous API.

If the underlying library has an async-ready API, all you need to do is add an Async suffix (or TaskAsync suffix) on the synchronous method name. For example, an Entity Framework call to First can be replaced with a call to FirstAsync. In some cases, you might want to use an alternative type. For example, HttpClient is a more async-friendly replacement for WebClient and HttpWebRequest. In some cases, you might need to upgrade the version of your library. Entity Framework, for example, acquired an async API in version 6.

Consider the code in Figure 1. This is a simple example with a service layer and some business logic. In this example, there’s only one low-level operation—retrieving a frob identifier string from a Web API in WebDataService.Get. This is the logical place to begin the asynchronous conversion. In this case, the developer can choose to either replace WebClient.DownloadString with WebClient.DownloadStringTaskAsync, or replace WebClient with the more async-friendly HttpClient.

The second step is to change the synchronous API call to an asynchronous API call, and then await the returned task. When code invokes an asynchronous method, it’s generally proper to await the returned task. At this point, the compiler will complain. The following code will cause a compiler error with the message, “The ‘await’ operator can only be used within an async method. Consider marking this method with the ‘async’ modifier and changing its return type to ‘Task<string>’”:

public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
The compiler guides you to the next step. Mark the method as async and change the return type. If the return type of the synchronous method is void, then the return type of the asynchronous method should be Task. Otherwise, for any synchronous method return type of T, the asynchronous method return type should be Task<T>. When you change the return type to Task/Task<T>, you should also modify the method name to end in Async, to follow the Task-Based Asynchronous Pattern guidelines. The following code shows the resulting method as an asynchronous method:

public sealed class WebDataService : IDataService
{
  public async Task<string> GetAsync(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
Before moving on, check the rest of this method for any other blocking or synchronous API calls that you can make async. Asynchronous methods shouldn’t block, so this method should call asynchronous APIs if they’re available. In this simple example, there are no other blocking calls. In real-world code, keep an eye out for retry logic and optimistic conflict resolution.

Entity Framework should get a special mention here. One subtle “gotcha” is lazy loading of related entities. This is always done synchronously. If possible, use additional explicit asynchronous queries instead of lazy loading.

Now this method is finally done. Next, move to all methods that reference this one, and follow this procedure again. In this case, WebDataService.Get was part of an interface implementation, so you must change the interface to enable asynchronous implementations:

public interface IDataService
{
  Task<string> GetAsync(int id);
}
Next, move to the calling methods and follow the same steps. You should end up with something like the code in Figure 2. Unfortunately, code won’t compile until all calling methods are transformed to async, and then all of their calling methods are transformed to async, and so on. This cascading nature of async is the burdensome aspect of brownfield development.

Figure 2 Change All Calling Methods to Async
public interface IDataService
{
  Task<string> GetAsync(int id);
}
public sealed class WebDataService : IDataService
{
  public async Task<string> GetAsync(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
public sealed class BusinessLogic
{
  private readonly IDataService _dataService;
  public BusinessLogic(IDataService dataService)
  {
    _dataService = dataService;
  }
  public async Task<string> GetFrobAsync()
  {
    // Try to get the new frob id.
    var result = await _dataService.GetAsync(17);
    if (result != string.Empty)
      return result;
    // If the new one isn't defined, get the old one.
    return await _dataService.GetAsync(13);
  }
}
Eventually, the level of asynchronous operation in your code base will grow until it hits a method that isn’t called by any other methods in your code. Your top-level methods are called directly by whichever framework you’re using. Some frameworks such as ASP.NET MVC permit asynchronous code directly. For example, ASP.NET MVC controller actions can return Task or Task<T>. Other frameworks such as Windows Presentation Foundation (WPF) permit asynchronous event handlers. So, for example, a button click event might be async void.

Hit the Wall

As the level of asynchronous code grows throughout your application, you might reach a point where it seems impossible to continue. The most common examples of this are object-oriented constructs, which don’t mesh with the functional nature of asynchronous code. Constructors, events and properties have their own challenges.

Rethinking the design is generally the best way around these difficulties. One common example is constructors. In the synchronous code, a constructor method might block on I/O. In the asynchronous world, one solution is to use an asynchronous factory method instead of a constructor. Another example is properties. If a property synchronously blocks on I/O, that property should probably be a method. An asynchronous conversion exercise is great at exposing these kinds of design issues that creep into your code base over time.

Transformation Tips

Performing an asynchronous code transformation can be scary the first few times, but it really becomes second nature after a bit of practice. As you feel more comfortable with converting synchronous code to asynchronous, here are a few tips you can start using during the conversion process.

As you convert your code, keep an eye out for concurrency opportunities. Asynchronous-concurrent code is often shorter and simpler than synchronous-concurrent code. For example, consider a method that has to download two different resources from a REST API. The synchronous version of that method would almost certainly download one and then the other. However, the asynchronous version could easily start both downloads and then asynchronously wait for both to complete using Task.WhenAll.

Another consideration is cancellation. Usually, synchronous application users are used to waiting. If the UI is responsive in the new version, they might expect the ability to cancel the operation. Asynchronous code should generally support cancellation unless there’s some other reason it can’t. For the most part, your own asynchronous code can support cancellation just by taking a CancellationToken argument and passing it through to the asynchronous methods it calls.

You can convert any code using Thread or BackgroundWorker to use Task.Run instead. Task.Run is far easier to compose than Thread or BackgroundWorker. For example, it’s much easier to express, “start two background computations and then do this other thing when they have both completed,” with the modern await and Task.Run, than with primitive threading constructs.

Vertical Partitions

The approach described so far works great if you’re the only developer for your application, and you have no issues or requests that would interfere with your asynchronous conversion work. That’s not very realistic, though, is it?

If you don’t have the time to convert your entire code base to be asynchronous all at once, you can approach the conversion with a slight modification called vertical partitions. Using this technique, you can do your asynchronous conversion to certain sections of code. Vertical partitions are ideal if you’d like to just “try out” asynchronous code.

To create a vertical partition, identify the user-level code you’d like to make asynchronous. Perhaps it’s the event handler for a UI button that saves to a database (where you’d like to keep the UI responsive), or a heavily used ASP.NET request that does the same (where you’d like to reduce the resources required for that specific request). Walk through the code, laying out the call tree for that method. Then you can start at the low-level methods and transform your way up the tree.

Other code will no doubt use those same low-level methods. Because you’re not ready to make all that code asynchronous, the solution is to create a copy of the method. Then transform that copy to be asynchronous. This way, the solution can still build at every step. When you’ve worked your way up to the user-level code, you’ll have created a vertical partition of asynchronous code within your application. A vertical partition based on our example code would appear as shown in Figure 3.

Figure 3 Use Vertical Partitions to Convert Sections of Code to Async
public interface IDataService
{
  string Get(int id);
  Task<string> GetAsync(int id);
}
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    using (var client = new WebClient())
      return client.DownloadString("http://www.example.com/api/values/" + id);
  }
  public async Task<string> GetAsync(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
public sealed class BusinessLogic
{
  private readonly IDataService _dataService;
  public BusinessLogic(IDataService dataService)
  {
    _dataService = dataService;
  }
  public string GetFrob()
  {
    // Try to get the new frob id.
    var result = _dataService.Get(17);
    if (result != string.Empty)
      return result;
    // If the new one isn't defined, get the old one.
    return _dataService.Get(13);
  }
  public async Task<string> GetFrobAsync()
  {
    // Try to get the new frob id.
    var result = await _dataService.GetAsync(17);
    if (result != string.Empty)
      return result;
    // If the new one isn't defined, get the old one.
    return await _dataService.GetAsync(13);
  }
}
You may have noticed there’s some code duplication with this solution. All the logic for the synchronous and asynchronous methods is duplicated, which isn’t good. In a perfect world, code duplication for this vertical partition is only temporary. The duplicated code would exist only in your source control until the application has been completely converted. At this point, you can remove all old synchronous APIs.

However, you can’t do this in all situations. If you’re developing a library (even one only used internally), backward compatibility is a primary concern. You might find yourself needing to maintain synchronous APIs for quite some time.

There are three possible responses to this situation. First, you could drive adoption of asynchronous APIs. If your library has asynchronous work to do, it should expose asynchronous APIs. Second, you could accept the code duplication as a necessary evil for backward compatibility. This is an acceptable solution only if your team has exceptional self-discipline or if the backward-compatibility constraint is only temporary.

The third solution is to apply one of the hacks outlined here. While I can’t really recommend any of these hacks, they can be useful in a pinch. Because their operation is naturally asynchronous, each of these hacks is oriented around providing a synchronous API for a naturally asynchronous operation, which is a well-known anti-pattern described in greater detail in a Server & Tools Blogs post at bit.ly/1JDLmWD.

The Blocking Hack

The most straightforward approach is to simply block the asynchronous version. I recommend blocking with GetAwaiter().GetResult instead of Wait or Result. Wait and Result will wrap any exceptions within an AggregateException, which complicates error handling. The sample service layer code would look like the code shown in Figure 4 if it used the blocking hack.

Figure 4 Service Layer Code Using the Blocking Hack
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    return GetAsync(id).GetAwaiter().GetResult();
  }
  public async Task<string> GetAsync(int id)
  {
    // This code will not work as expected.
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
Unfortunately, as the comment implies, that code wouldn’t actually work. It results in a common deadlock described in my “Best Practices in Asynchronous Programming” article I mentioned earlier.

This is where the hack can get tricky. A normal unit test will pass, but the same code will deadlock if called from a UI or ASP.NET context. If you use the blocking hack, you should write unit tests that check this behavior. The code in Figure 5 uses the Async­Context type from my AsyncEx library, which creates a context similar to a UI or ASP.NET context.

Figure 5 Use the AsyncContext Type
[TestClass]
public class WebDataServiceUnitTests
{
  [TestMethod]
  public async Task GetAsync_RetrievesObject13()
  {
    var service = new WebDataService();
    var result = await service.GetAsync(13);
    Assert.AreEqual("frob", result);
  }
  [TestMethod]
  public void Get_RetrievesObject13()
  {
    AsyncContext.Run(() =>
    {
      var service = new WebDataService();
      var result = service.Get(13);
      Assert.AreEqual("frob", result);
    });
  }
}
The asynchronous unit test passes, but the synchronous unit test never completes. This is the classic deadlock problem. The asynchronous code captures the current context and attempts to resume on it, while the synchronous wrapper blocks a thread in that context, preventing the asynchronous operation from completing.

In this case, our asynchronous code is missing a ConfigureAwait­(false). However, the same problem can be caused by using WebClient. WebClient uses the older event-based asynchronous pattern (EAP), which always captures the context. So even if your code uses ConfigureAwait(false), the same deadlock will occur from the WebClient code. In this case, you can replace WebClient with the more async-friendly HttpClient and get this to work on the desktop, as shown in Figure 6.

Figure 6 Use HttpClient with ConfigureAwait(false) to Prevent Deadlock
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    return GetAsync(id).GetAwaiter().GetResult();
  }
  public async Task<string> GetAsync(int id)
  {
    using (var client = new HttpClient())
      return await client.GetStringAsync(
      "http://www.example.com/api/values/" + id).ConfigureAwait(false);
  }
}
The blocking hack requires your team to have strict discipline. They need to ensure ConfigureAwait(false) is used everywhere. They must also require all dependent libraries to follow the same discipline. In some cases, this just isn’t possible. As of this writing, even HttpClient captures the context on some platforms.

Another drawback to the blocking hack is it requires you to use ConfigureAwait­(false). It’s simply unsuitable if the asynchronous code actually does need to resume on captured context. If you do adopt the blocking hack, you’re strongly recommended to perform unit tests using AsyncContext or another similar single-threaded context to catch any lurking deadlocks.

The Thread Pool Hack

A similar approach to the Blocking Hack is to offload the asynchronous work to the thread pool, then block on the resulting task. The code using this hack would look like the code shown in Figure 7.

Figure 7 Code for the Thread Pool Hack
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    return Task.Run(() => GetAsync(id)).GetAwaiter().GetResult();
  }
  public async Task<string> GetAsync(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
The call to Task.Run executes the asynchronous method on a thread pool thread. Here it will run without a context, thus avoiding the deadlock. One of the problems with this approach is the asynchronous method can’t depend on executing within a specific context. So, it can’t use UI elements or the ASP.NET HttpContext.Current.

Another more subtle “gotcha” is the asynchronous method may resume on any thread pool thread. This isn’t a problem for most code. It can be a problem if the method uses per-thread state or if it implicitly depends on the synchronization provided by a UI context.

You can create a context for a background thread. The AsyncContext type in my AsyncEx library will install a single-threaded context complete with a “main loop.” This forces the asynchronous code to resume on the same thread. This avoids the more subtle “gotchas” of the thread pool hack. The example code with a main loop for the thread pool thread would look like the code shown in Figure 8.

Figure 8 Use a Main Loop for the Thread Pool Hack
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    var task = Task.Run(() => AsyncContext.Run(() => GetAsync(id)));
    return task.GetAwaiter().GetResult();
  }
  public async Task<string> GetAsync(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
Of course, there’s a disadvantage to this approach, as well. The thread pool thread is blocked within the AsyncContext until the asynchronous method completes. This blocked thread is there, as well as the primary thread calling the synchronous API. So, for the duration of the call, there are two threads being blocked. On ASP.NET in particular, this approach will significantly reduce the application’s ability to scale.

The Flag Argument Hack

This hack is one I haven’t used yet. It was described to me by Stephen Toub during his tech review of this article. It’s a great approach and my favorite of all these hacks.

The flag argument hack takes the original method, makes it private, and adds a flag to indicate whether the method should run synchronously or asynchronously. It then exposes two public APIs, one synchronous and the other asynchronous, as shown in Figure 9.

Figure 9 Flag Argument Hack Exposes Two APIs
public interface IDataService
{
  string Get(int id);
  Task<string> GetAsync(int id);
}
public sealed class WebDataService : IDataService
{
  private async Task<string> GetCoreAsync(int id, bool sync)
  {
    using (var client = new WebClient())
    {
      return sync
        ? client.DownloadString("http://www.example.com/api/values/" + id)
        : await client.DownloadStringTaskAsync(
        "http://www.example.com/api/values/" + id);
    }
  }
  public string Get(int id)
  {
    return GetCoreAsync(id, sync: true).GetAwaiter().GetResult();
  }
  public Task<string> GetAsync(int id)
  {
    return GetCoreAsync(id, sync: false);
  }
}
The GetCoreAsync method in this example has one important property—if its sync argument is true, it always returns an already-­completed task. The method will block when its flag argument requests synchronous behavior. Otherwise, it acts just like a normal asynchronous method.

The synchronous Get wrapper passes true for the flag argument and then retrieves the result of the operation. Note there’s no chance of a deadlock because the task is already completed. The business logic follows a similar pattern, as shown in Figure 10.

Figure 10 Apply Flag Argument Hack to Business Logic
public sealed class BusinessLogic
{
  private readonly IDataService _dataService;
  public BusinessLogic(IDataService dataService)
  {
    _dataService = dataService;
  }
  private async Task<string> GetFrobCoreAsync(bool sync)
  {
    // Try to get the new frob id.
    var result = sync
      ? _dataService.Get(17)
      : await _dataService.GetAsync(17);
    if (result != string.Empty)
      return result;
    // If the new one isn't defined, get the old one.
    return sync
      ? _dataService.Get(13)
      : await _dataService.GetAsync(13);
  }
  public string GetFrob()
  {
    return GetFrobCoreAsync(sync: true).GetAwaiter().GetResult();
  }
  public Task<string> GetFrobAsync()
  {
    return GetFrobCoreAsync(sync: false);
  }
}
You do have the option of exposing the CoreAsync methods from your service layer. This simplifies the business logic. However, the flag argument method is more of an implementation detail. You’d need to weigh the advantage of cleaner code against the disadvantage of exposing implementation details, as shown in Figure 11.  The advantage of this hack is the logic of the methods stays basically the same. It just calls different APIs based on the value of the flag argument. This works great if there’s a one-to-one correspondence between synchronous and asynchronous APIs, which is usually the case.

Figure 11 Implementation Details Are Exposed, but the Code Is Clean
public interface IDataService
{
  string Get(int id);
  Task<string> GetAsync(int id);
  Task<string> GetCoreAsync(int id, bool sync);
}
public sealed class BusinessLogic
{
  private readonly IDataService _dataService;
  public BusinessLogic(IDataService dataService)
  {
    _dataService = dataService;
  }
  private async Task<string> GetFrobCoreAsync(bool sync)
  {
    // Try to get the new frob id.
    var result = await _dataService.GetCoreAsync(17, sync);
    if (result != string.Empty)
      return result;
    // If the new one isn't defined, get the old one.
    return await _dataService.GetCoreAsync(13, sync);
  }
  public string GetFrob()
  {
    return GetFrobCoreAsync(sync: true).GetAwaiter().GetResult();
  }
  public Task<string> GetFrobAsync()
  {
    return GetFrobCoreAsync(sync: false);
  }
}
It may not work as well if you want to add concurrency to your asynchronous code path, or if there’s not an ideal corresponding asynchronous API. For example, I would prefer to use HttpClient over WebClient in WebDataService, but I’d have to weigh that against the added complexity it would cause in the GetCoreAsync method.

The primary disadvantage of this hack is flag arguments are a well-known anti-pattern. Boolean flag arguments are a good indicator a method is really two different methods in one. However, the anti-pattern is minimized within the implementation details of a single class (unless you choose to expose your CoreAsync methods). Despite this, it’s still my favorite of the hacks.

The Nested Message Loop Hack

This final hack is my least favorite. The idea is that you set up a nested message loop within the UI thread and execute the asynchronous code within that loop. This approach isn’t an option on ASP.NET. It may also require different code for various UI platforms. For example, a WPF application could use nested dispatcher frames, while a Windows Forms application could use DoEvents within a loop. If the asynchronous methods don’t depend on a particular UI platform, you can also use AsyncContext to execute a nested loop, as shown in Figure 12.

Figure 12 Execute a Nested Message with AsyncContext
public sealed class WebDataService : IDataService
{
  public string Get(int id)
  {
    return AsyncContext.Run(() => GetAsync(id));
  }
  public async Task<string> GetAsync(int id)
  {
    using (var client = new WebClient())
      return await client.DownloadStringTaskAsync(
      "http://www.example.com/api/values/" + id);
  }
}
Don’t be deceived by the simplicity of this example code. This hack is the most dangerous of them all, because you must consider reentrancy. This is particularly true if the code uses nested dispatcher frames or DoEvents. In that case, the entire UI layer must now handle unexpected reentrancy. Reentrant-safe applications require a considerable amount of careful thought and planning.

Wrapping Up

In an ideal world, you could perform a relatively simple code transformation from synchronous to asynchronous and everything would be rainbows and unicorns. In the real world, it’s often necessary for synchronous and asynchronous code to coexist. If you just want to try out async, create a vertical partition (with code duplication) until you’re comfortable using async. If you must maintain the synchronous code for backward-compatibility reasons, you’ll have to live with the code duplication or apply one of the hacks.

Someday, asynchronous operations will only be represented with asynchronous APIs. Until then, you have to live in the real world. I hope these techniques will help you adopt async into your existing applications in a way that works best for you.

Stephen Cleary is a husband, father and programmer living in northern Michigan. He has worked with multithreading and asynchronous programming for 16 years and has used async support in the Microsoft .NET Framework since the first CTP. Follow his projects and blog posts at stephencleary.com.

Thanks to the following Microsoft technical experts for reviewing this article: James McCaffery and Stephen Toub
