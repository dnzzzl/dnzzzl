+++
date = '2026-02-23T13:25:31-04:00'
draft = true
title = 'C# Interview Guide'
+++

The following are common interview questions and important concepts to know when aspiring to hold a senior or advanced position in a software engineering team using C# and .NET.

## Value types vs reference types

Quick cheatsheet:

```
Value types include:
- int
- long
- float
- char
- bool
- struct

Reference types include:
- string
- class instances
- objects
```

Here is the real mental model beyond the naive and intuitive explanation of "Value types hold data while reference types point to an address": It has to do with what happens layers below in memory, when instantiating and assigning one to another, the data either lives in the __stack__ or the __heap__.

### The Stack

"The stack is a region of memory that is used to store local variables and function call information. It is called a 'stack' because it behaves like a stack of items, with the most recently added item being the first one to be removed (last in, first out)." [medium](https://medium.com/@AIbatros/c-stack-vs-heap-memory-f8a737af9919)

Here, value types hold the bytes of the assigned data directly. When assigning a value type to another variable, a copy of the actual value is made. These instances are independent of each other:

```csharp
int a = 5;
int b = a; // Copy is made
b = 10;    // a remains 5
```

Copies are made in most instances: assignment, method arguments, indexing, or storing in collections:

```csharp
void ModifyInt(int value) { value = 20; } // Copy passed, original unchanged

int original = 5;
ModifyInt(original);
Console.WriteLine(original); //original still 5
```

```csharp
List<int> list = new();
int i = 5;
list.Add(i); // Copy stored in list
i = 10;      // List still contains 5
```

### The Heap and your garbage: IDisposable, using and finalizers

"In C#, objects are dynamically allocated on the heap using the new keyword. When an object is no longer needed, it is the responsibility of the garbage collector to deallocate the memory and reclaim it for future use.

The heap is a more flexible region of memory than the stack, but it is also slower to allocate and deallocate memory from. This is because the heap has no fixed size and the garbage collector must constantly monitor and manage the memory being used." borrowed from this [medium](https://medium.com/@AIbatros/c-stack-vs-heap-memory-f8a737af9919) article.

The garbage collector is a background thread managed by the .NET runtime that, according to its own mysterious devices (for our current purposes at least), eventually frees up memory by disposing of objects so long as they are managed by the runtime itself in the first place, these are what we call managed objects.

Unmanaged objects as the counterpart to managed objects are resources that as developers we instantiate, use, and eventually stop using, but exist outside of .NET runtime's domain, such as a network connection, an opened file access, a bitmap. These are lower level resources from the OS that we use in our .NET code, as such they are unmanaged and it is our duty to use the tools given to us in the .NET framework to free up their allocated memory when their lifetime is over.

The first line of defense is the `using` keyword

```csharp
using (new resource){
  resource.UseMyResource()

} //at the end resource.Dispose() is called!
```

It is used in combination with that Dispose() method which comes from the IDisposable interface, already built into managed async objects, but for our classes which contain unmanaged properties, it will be our task to inherit from IDisposable, implement this Dispose method and tell the garbage collector people how to deal with our object reference once the data is no longer needed.

Similarly finalizers are the counterpart to constructors, these are  class methods called by the Garbage Collection thread when the object reference has been marked for disposal. A common pattern is to call the dispose method from this finalizer, this way whether Dispose() is responsibly called from our code or Garbage Collection deals with it on its own schedule, that Dispose() method will be called.

However this is where our first common pitfall and rule of thumb come into play: Dispose() should be idempotent, lest we have the GC execute the deconstructor `~MyClass(){Dispose();}`, which calls the dispose method and try to dereference objects that have already been dereferenced before.
The way we can achieve this is by declaring a bool argument that more or less reads Dispose(bool itIsSafeToDisposeOfManagedObjects). That way the finalizer can be setup to call it with `false` since the garbage collector is out doing its job anyways and the managed objects are not guaranteed to be there. While other parts of our code can call it with a value of `true` when we dispose proactively.

## Async != concurrency, Tasks, Threads, await, deadlocks and other asynchronous topics

C# introduced async-await keywords in 2010, since then this pattern of easy asynchronous programming has become a staple, even outside C#. It's easy enough to use but of course, complex underneath, let's take our first peek and try to understand.

The first thing to note is that the difference between our main function being preceded by `async` and not, is simple enough in high level C# land, but underneath, if we take a peek into the intermediate language in CLR-land, we see that the compiler completely rewrites it into a state machine at compile time.

There's no "async" IL instruction, it's all compile magic that transforms our readable code into stateful callbacks under the hood.

What Actually Happens:

  1. The method becomes a struct that implements IAsyncStateMachine
  2. Local variables → fields on that struct
  3. Each await creates a new state (State 0, 1, 2, etc.)
  4. MoveNext() drives execution forward when tasks complete

__State Transitions Example__

```csharp
  async Task MyMethod()
  {
      var result1 = await GetData(); // State 0: await here
      var result2 = await Process(result1); // State 1: await here
      return result2; // State 2: complete
  }
```

Compiles to roughly:

```csharp
  struct MyMethodStateMachine : IAsyncStateMachine
  {
      int state;
      Task<Data> awaiter1;
      Data result1;
      // ... and so on

      void MoveNext()
      {
          try
          {
              switch(state)
              {
                  case -1: //Initial
                      awaiter1 = GetData();
                      if  (!awaiter1.IsCompleted)
                      {
                          state = 0;
                          //Schedule continuation via awaiter
                          return;
                      }
                      goto case 0;
                  case 0: // After first await
                      result1 = awaiter1.GetResult();
                      awaiter2 = Process(result1);
                      if (!awaiter2.IsCompleted)
                      {
                          state = 1;
                          return;
                      }
                      goto case 1;

                  case 1: // After second await
                      return awaiter2.GetResult();
              }
          }
          catch { /* handle exceptions */ }
      }
  }
```

Here we see the MoveNext() function, its switch statement and what this actually looks like, it was helpful for me to take a look at the IL representation of the code.

__Practical knowledge:__

- Task.Run creates a separate thread through the Thread Pool, mainly used for CPU-bound background tasks that actually do need to hold up a thread.

- async/await: compiler magic that runs delegates, and manages resumption dynamically. It does introduce overhead and performance gains are a deeper subject to study at this point. It shines for tasks like network calls and IO bound work, where the thread does not need to actively work, but instead the work is delegated to the OS and execution of our async function will resume after a callback of completion.

This lends us nicely to talk about asynchrony vs concurrency. The quick breakdown is that asynchronous tasks are not necessarily concurrent, but concurrent tasks have to be executed asynchronously.

### CancellationToken

A cancellationToken is a struct that carries a signal across threads, it can produce an exception and signals for cancellation of a long running background task.
The pattern is similar to a factory pattern where a CancellationTokenSource is used to produce a cancellation token and subsequently can signal to async tasks using the token.

```
using System.Threading;
using System.Threading.Tasks;

async Task<string> GetUserAsync(int id, CancellationToken cancellationToken)
{
    await Task.Delay(200, cancellationToken); // pass token directly to Delay
    cancellationToken.ThrowIfCancellationRequested();
    return $"User-{id}";
}

async Task Main()
{
    var cts = new CancellationTokenSource(100); // cancels after 100ms
    try
    {
        var user = await GetUserAsync(1, cts.Token);
        Console.WriteLine(user);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Cancelled.");
    }
}
```

Two parts to this: setup and execution.

We setup our Async function to take in a CancellationToken and pass it along to any internal Async calls. Here we also check for cancellations with cancellationToken.ThrowIfCancellationRequested() .

Then in our calling code we create said Token through a cts CancellationTokenSource and catch for OperationCanceledException in case the task is cancelled.

### ValueTask

The problem to solve: using Task always stores an object to the heap. In high throughput code and cache situations, this may be expensive or unnecesary.
ValueTask is a mechanism that combines well with caching, where we may have two paths to a function: a synchronous cached path and an asynchronous call to network or db if the cache misses. ValueTask is a struct that allows us to return synchronously if the value is immediately available, and it does fallback to create a Task if the value does need to be awaited

```csharp
public ValueTask<string> GetAsync(string key)
{
    if (_cache.TryGetValue(key, out var val))
        return new ValueTask<string>(val); // zero allocation

    return new ValueTask<string>(FetchFromDbAsync(key)); // wraps Task
}
```

And that's how you use it.

## Memory<T> vs Span<T>

## References

- <https://medium.com/@AIbatros/c-stack-vs-heap-memory-f8a737af9919>
- <https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines>
