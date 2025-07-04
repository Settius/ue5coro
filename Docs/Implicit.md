# Implicit awaiters

Implicit awaiters get created when a coroutine awaits some kind of engine type
that's supported by this feature.
The coroutine does not get access to the awaiter itself: it cannot be created
in advance, or stored in a variable.

Aggregate awaiters (WhenAny, WhenAll, etc.) that normally expect awaitables as
parameters will accept these types directly.

### TCoroutine\<T\>

TCoroutine is not an awaiter, but it is directly awaitable.
Awaiting one from an async or latent coroutine results in different behavior:

* Async coroutines will immediately resume when the coroutine completes, on the
  same thread that it completed on.
* Latent coroutines can only await other coroutines on the game thread.

In either case, awaiting a coroutine that has already completed is instant, and
synchronously continues on the caller thread.

The await expression will result in T.
If the awaited TCoroutine was a non-const rvalue and T is movable, then this
value will have been moved from the coroutine's return value storage; otherwise,
it is a copy.
Awaiting TCoroutine\<\> (T=void) results in no value, and its lvalue/rvalue
awaits behave identically.

Example:
```cpp
using namespace UE5Coro;

TCoroutine Coro1 = []() -> TCoroutine<int>
{
    co_return 10;
}();
TCoroutine Coro2 = [](TCoroutine<int> C) -> TCoroutine<>
{
    int Ten = co_await C;
}(Coro1);
```

### UE::Tasks::TTask\<T\>

Awaiting an incomplete TTask will move the coroutine's execution into the
UE::Tasks system (notably, this means it's no longer on the game thread) after
the given task has completed.
If the TTask is already complete, the coroutine continues synchronously on the
same thread, as an optimization.

The await expression will result in T&, not T, matching TTask\<T\>::GetResult().

Example:
```cpp
using namespace UE::Tasks;
using namespace UE5Coro;

TCoroutine<> Example(TTask<int> Task)
{
    int& Value1 = co_await Task;
    // Task returning a reference doesn't mean you have to use one
    int Value2 = co_await Task;
}
```

### TFuture\<T\>

> [!WARNING]
> TFuture's API is unstable in the engine itself; it is not recommended for use.

Awaiting a TFuture consumes it, and resumes the coroutine once it has completed
on the same thread that would be used by TFuture::Then or Next.
The coroutine will continue synchronously on its current thread if the future
was ready at the time of the co_await.

Because this is a destructive operation, the future **must** be an rvalue.
Move the future into the await expression if needed.
TSharedFuture\<T\> is not supported due to engine limitations.

The await expression will result in T, which will have been moved from the
future instead of copied, if possible.
If T is a reference, the result will be the reference to the same object that
the TFuture refers to: copying or moving does not apply.

Awaiting TFuture\<void\> results in void, not the meaningless int value that
TFuture normally provides.

Example:
```cpp
using namespace UE5Coro;

TCoroutine<> Example(TPromise<int>& Promise, TFuture<int> Future)
{
    int Value1 = co_await Promise.GetFuture(); // OK, already an rvalue
 // int Value2 = co_await Future; will not compile
    int Value2 = co_await std::move(Future); // OK, moved into the co_await
}
```

### Delegates

Awaiting a delegate resumes the coroutine when that delegate is next Execute()d
or Broadcast().
Doing so Binds or Adds to the delegate, and Unbinds/Removes the binding when the
coroutine resumes.

> [!NOTE]
> To await an engine function that expects an already-bound delegate, use
> [Async::Chain](AsyncChain.md).

The following engine delegates are supported, including any number of parameters,
with or without return values:
* TDelegate (DECLARE_DELEGATE, ~~DECLARE_TS_DELEGATE~~[^ts])
* TMulticastDelegate (DECLARE_MULTICAST_DELEGATE, DECLARE_TS_MULTICAST_DELEGATE)
* Types created by DECLARE_DYNAMIC_DELEGATE
* Types created by DECLARE_DYNAMIC_MULTICAST_DELEGATE
* Types created by DECLARE_DYNAMIC_MULTICAST_SPARSE_DELEGATE
* Types created by the deprecated DECLARE_EVENT

[^ts]: Unreal does not have non-multicast DECLARE_TS_DELEGATE macros, but the
       TDelegates that they would logically expand to are supported anyway.

Parameters must be _MoveConstructible_.
Return types must be _DefaultConstructible_ or void.
More than nine parameters are supported, and interacting with DYNAMIC delegates
this way does not require a UFUNCTION or even a UCLASS at all.

> [!CAUTION]
> Thread safety and synchronization is your responsibility: there are no checks
> or other measures taken against data races when the await expression starts
> (Bind/Add) or finishes (Unbind/Remove/UObject invalidation).
>
> The coroutine being resumed again after it has unbound from the delegate is
> undefined behavior.
> This can occur if something copies a non-DYNAMIC delegate while it's bound,
> then uses the copy.
>
> Similarly, coroutine cancellation itself is thread safe, but most Unreal
> delegates are not.
> A coroutine awaiting a delegate will unbind on the thread that triggered the
> delegate, or the thread that cancels the coroutine, whichever occurs first.

The delegate will directly and synchronously call into the coroutine.
If the delegate is destroyed or isn't ever invoked, the coroutine will not be
resumed, which could result in a memory leak.
A delegate getting destroyed while it's being awaited is undefined behavior.

Awaiting delegates supports expedited cancellation.
Canceling the TCoroutine will prevent the memory leak.

The await expression results in an unspecified type that may be used with
structured bindings to optionally receive the delegate's parameters.
Reference parameters are passed through as references, and may be written to.
These writes will go through the delegate call into the original referenced
variables.

The await expression's value may be safely discarded if the coroutine does not
want to receive the parameters.
Using the return value of the await expression in any way that's not part of a
structured binding declaration (including storing the entirety of it in an
`auto` local variable) is not supported.

The validity of references and pointers depend on the delegate's caller, but
even the shortest-lived references will be valid until the next co_await or
co_return.

For delegates with non-void return types, a default-constructed value is
returned to the caller at the coroutine's next co_await or co_return.
Even if the coroutine has a different co_return result, the delegate will
receive the default-constructed value.
The delegate's return type and the coroutine's result type are independent.

Example:
```cpp
using namespace UE5Coro;

class FExample
{
    TDelegate<FString(FName, bool, int&)> Delegate;

    TCoroutine<int> Foo()
    {
        UE_LOGFMT(LogTemp, Display, "First");
        auto&& [Name, bSomething, OutValue] = co_await Delegate;
        OutValue = 1;
        UE_LOGFMT(LogTemp, Display, "Third");
        co_return 0;
    }

public:
    void Bar()
    {
        Foo();
        UE_LOGFMT(LogTemp, Display, "Second");
        int Value = 0;
        FString EmptyString = Delegate.Execute("Name", true, Value);
        // Value == 1
        UE_LOGFMT(LogTemp, Display, "Fourth");
    }
};
```

#### Generic workarounds

To work with callback-based functions that aren't supported by
[Async::Chain](AsyncChain.md), or the delegate co_await feature described above,
or when the suitability or safety of these is in doubt,
[FAwaitableEvent](Threading.md#fawaitableevent) can be used as part of a
generic, thread-safe workaround.

If it is being awaited when the callback is invoked, `Trigger()` will call back
into the coroutine synchronously; otherwise, it will let the coroutine through
synchronously if the callback has already happened.

The following technique should work for nearly everything that supports lambdas:

```c++
using namespace UE5Coro;

TCoroutine<> Example()
{
    FAwaitableEvent Event;

    int Data1, Data2;
    imaginary_library_delegate_t<int, int> Delegate([&](int Param1, int Param2)
    {
        Data1 = Param1; // FAwaitableEvent itself is thread safe,
        Data2 = Param2; // but you're still responsible for these two!
        Event.Trigger();
    });
    some_imaginary_library_function(Delegate);
    // Directly-passed lambdas are more or less the same:
    // another_imaginary_library_function([&](int Param1, int Param2){...});
    co_await Event;
    // Data1 and Data2 are known to be valid here
}
```

This complex example demonstrates using `shared_ptr` with an interface-like type
that needs to be subclassed, and the `co_await` being optional:

```c++
using namespace UE5Coro;

TCoroutine<> Example()
{
    struct FMyListener final : imaginary_library_listener_t
    {
        std::shared_ptr<FAwaitableEvent> Event;
        virtual void execute_callback() override { Event->Trigger(); }
    };

    // FAwaitableEvent is immovable and non-copyable, but shared_ptr isn't
    std::shared_ptr<FAwaitableEvent> Event = std::make_shared<FAwaitableEvent>();
    FMyListener Listener;
    Listener.Event = Event;
    some_imaginary_library_function(std::move(Listener));

    if (bSomething)
        co_await *Event;

    // Even if !bSomething, the shared_ptr keeps the event alive here, but
    // depending on the semantics of some_imaginary_library_function, Listener
    // going out of scope when this coroutine ends might need explicit handling.
}
```

Many C libraries take raw function pointers, precluding lambda captures, but
they often support specifying a custom `void*` that will be passed back to it:

```c++
using namespace UE5Coro;

TCoroutine<> Example()
{
    FAwaitableEvent Event;
    void (*Fn)(void*) = [](void* UserData)
    {
        static_cast<FAwaitableEvent*>(UserData)->Trigger();
    };

    some_imaginary_c_library_function(Fn, &Event);
    co_await Event; // The unconditional co_await keeps &Event valid here
    // Unregister the function pointer from the C library here, if required
}
```
