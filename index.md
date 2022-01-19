# Async programming

## Terminology

Before we jump into the story, I'll go over some of the terms we'll be using.

- CPU-Threads, a thread that's managed by the CPU
- OS-Threads, a thread that's managed by the OS
- Green threads, a thread that's managed by the runtime
- Threads, just any form of thread, can be a CPU-Thread, OS-thread or a green-thread
- Concurrently, multiple threads running after each-other
- Parallelism, multiple threads running at the same time
- Multithreaded, multiple threads running concurrently (or in parallel)
- Asynchronous
- Data streams

## The issue, utilization of the CPU

CPU's have multiple cores, and multiple threads nowadays. Constantly, threads block resulting in the underlaying core
doing nothing. Or cores do nothing, since there's no task for them to do.

And that's a waste, they should be used as much as possible, as effectively as possible. Let's start at concurrent
programming, utilizing a thread as much as possible by switching tasks when one tasks blocks, which is where the
await-async keywords come into the story.

## Solution 1, Await-async keywords

Grab your coloring book kids! We're gonna start coloring our methods.

We've 2 types, async and sync methods. You make a method async by using the async method This looks really simple, and
like an amazing solution. But there's a couple of caveats.

1. You can only call async methods in async methods, you can't call an async method in a sync method.
2. Sync functions return a value, async ones return a wrapper around the value (Future, Promise)
3. You can't use try-catch, since the async one happens in a callback. (Callback hell)

To elaborate on this, take a view at this beautiful callback hell.

*JavaScript*

```js
randomUser(function (user) {
    randomServer(function (server) {
        server.getMember(user, function (member) {
            if (member.hasAcceptedRules) {
                member.giveRole(Role.STARTER, function (action) {
                    if (action.hadSuccess) {
                        user.sendMessage("Gave you the starter role!");
                    }
                });
            } else {
                user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
            }
        })
    })
});
```

There is a ton of callbacks in this code, which makes it really hard to read. That's why people came up with the `await`
keyword.

### The `await` keyword

The `await` keyword allows people to wait for the method to finish, removing the need of callback. This fixes that sync
methods can't call async methods, but still numerous issues remain. An example of with the `await` keyword can be found
below.

*JavaScript*

```js
var user = await randomUser();
var server = await randomServer();
var member = await server.getMember(user);

if (member.hasAcceptedRules()) {
    var action = await member.giveRole(Role.STARTER);

    if (action.hadSuccess()) {
        user.sendMessage("Gave you the starter role!");
    }
} else {
    user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
}
 ```

Notice how much easier it is to read, while it's still not perfect, it is a lot easier. The `await` keyword still is
annoying, you've to constantly add it.

This removed 2 of our issues, callback hell is non-existent in this case, and you can call async methods from sync
methods now. However, there still are numerous of issues, so let's take a look at those.

1. Sync functions return values, while async return a Task\<T\> (a wrapper of the value)
2. Async methods only return the value if you use await, which means your method has to become `async` too
3. Constant await gibberish, which floods the code
4. Async methods, are still *async*

So yeah, is this *really* a good solution? I'd say no, it doesn't feel polished, it's not that pleasant to write either.

## Solution 2, threads

So, how about instead using concurrency we use parallelism? Sure! Threads, you can come in now.

Threads, OS-Threads to be specific, using those we can let our processor run multiple things at the same time. You're
loading a file? Just creete a new thread and do it on there!

At least, you'd want to do that. But it's not that simple, threads aren't lightweight. Thread's are heavy. OS-threads
are designed with everything in mind, so they store a lot more data than they require.

So a solution, you use a pool, a ThreadPool. A thread pool re-uses threads, this way you don't constantly create new
threads, and only create new ones when it's actually needed. So you achieve parallelism, but still it's not, perfect you
know? It doesn't feel nice, to constantly access a ThreadPool.

You made it run in parallel, but not concurrent. Everytime your thread gets blocked the OS needs to handle it, which
happens rather inefficient.

And there also are security issues, a Thread exposes it's thread locals. A thread local is a variable local to the
Thread, once set they cannot be removed unless overwritten by a different value. While this makes sense, when in the
context of a ThreadPool, this means that all tasks using the Thread also get access to the locals.

And last, but not least, due to the fact it's an OS thread, the cancellation program is rather complex and expensive. To
elaborate on this, if a thread calls Thread#interrupt on another thread, this thread sets the status to Interrupted, and
throws a InterruptedException to all blocking methods. After which the status gets cleared.

The status get's cleared for 2 reasons. First of all, this allows ThreadPool's to re-use the Thread. Second of all, one
might require some of the Thread's blocking methods after an error.

While the purpose of this behaviour, makes sense. It's just rather error prone, preferably we'd want to create a new
Thread instead.

*Java*

```java
public class Application {
    public static void main(String[] args) {
        new Thread(() -> {
            var user = randomUser();
            var server = randomServer();
            var member = server.getMember(user);

            if (member.hasAcceptedRules()) {
                var action = member.giveRole(Role.STARTER);

                if (action.hadSuccess()) {
                    user.sendMessage("Gave you the starter role!");
                }
            } else {
                user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
            }
        });
    }
}
```

But at least, the code is rather readable, no await, no playing around with Task<T>. This runs in parallel, the
concurrency is still being handled by the OS, ineffectively.

It'd be awesome if we somehow, mixed those together in an effective manner? Which is where reactive comes in!

## Solution 3, (Functional) reactive programming

Reactive combines the best and worst of both, it results in a concurrent and parallel method of programming.

To take a look at the definition according to Wikipedia
> Functional reactive programming (FRP) is a programming paradigm for reactive programming (asynchronous dataflow programming) using the building blocks of functional programming (e.g. map, reduce, filter).

As you might notice, this sounds rather advanced. And unfortunately I'll have to tell you, reactive programming has a
rather steep learning curve. 

So to elaborate on what reactive programming is, I'll show you an example which we will break down together.

```java
    Stream.of
```