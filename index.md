## Terminology

Before we jump into the story, I'll go over some terms we'll be using.

- CPU-Threads, a thread that's managed by the CPU
- OS-Threads, a thread that's managed by the OS
- Green threads, a thread that's managed by the runtime
- Threads, just any form of thread, can be a CPU-Thread, OS-thread or a green-thread
- Concurrently, multiple threads running after each-other
- Parallelism, multiple threads running at the same time
- Multithreaded, multiple threads running concurrently (or in parallel)
- Callback, any executable code that is passed as an argument to other code
- Asynchronous, the occurrence of events independent of the main program flow and ways to deal with such events
- Data streams, a sequence of elements made available over time

## The issue, utilization of the CPU

CPU's have multiple cores, and multiple (CPU-)threads nowadays. Constantly, threads block resulting in the underlying
core doing nothing. Or the cores weren't doing anything in the first place, since there isn't a task for them.

And that's a waste, they should be used as much as possible, as effectively as possible. So let's start at concurrent
programming, utilizing a thread as much as possible by switching tasks when one tasks blocks, which is where the
await-async keywords come into the story.

## Solution 1, Await-async keywords

Grab your coloring book kids! We're going to start coloring our methods.

We've 2 types, async and sync methods. You make a method async by using the `async` keyword. This looks really simple,
and like an amazing solution. But there's a couple of caveats.

1. You can only call async methods in async methods, you can't call an async method in a sync method. (unless using a
   callback)
2. Sync functions return a value, async one's return a wrapper around the value (Future, Promise)
3. You can't use try-catch, since the async one happens on another thread. (Often in callbacks)
4. When using callbacks from code that you never wrote, you have to *trust* them that the callback actually runs.

To elaborate on this, take a view at this beautiful callback hell.

*JavaScript*

```js
randomServer(function (server) {
    server.getRandomMember(function (member) {
        if (member.hasAcceptedRules) {
            member.giveRole(Role.STARTER, function (action) {
                if (action.hadSuccess) {
                    user.sendMessage("Gave you the starter role!");
                } else {
                    user.sendMessage("Something went wrong, please contact a developer)");
                }
            });
        } else {
            user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
        }
    })
});
```

There is a ton of callbacks in this code, which makes it really hard to read. It goes inside a method, inside a method,
inside another method. Can't forget that debugging will be less enjoyable in this case either.

Which is why people came up with the `await` keyword.

### The `await` keyword

The `await` keyword allows people to wait for the method to finish, removing the need of callback. This fixes that sync
methods can't call async methods (without a callback), but still numerous issues remain. An example of with the `await`
keyword can be found below.

*JavaScript*

```js
var server = await randomServer();
var member = await server.getRandomMember();

if (member.hasAcceptedRules()) {
    var action = await member.giveRole(Role.STARTER);

    if (action.hadSuccess()) {
        user.sendMessage("Gave you the starter role!");
    } else {
        user.sendMessage("Something went wrong, please contact a developer)");
    }
} else {
    user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
}
 ```

Notice how much easier it is to read, by all means it's still not perfect, it is a lot easier. The `await` keyword still
is annoying, you've to constantly add it.

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
loading a file? Just create a new thread and do it on there!

At least, you'd want to do that. But it's not that simple, threads aren't lightweight. Threads are heavy. OS-threads
are designed with everything in mind, so they store a lot more data than they require.

So a solution, you use a pool, a ThreadPool. A thread pool re-uses threads, this way you don't constantly create new
threads, and only create new ones when it's actually needed. So you achieve parallelism, but still it's not, perfect you
know? It doesn't feel nice, to constantly access a ThreadPool.

You made it run in parallel, but not concurrent. Everytime your thread gets blocked the OS needs to handle it, which
happens rather inefficient.

And there also are security issues, a Thread exposes its thread locals. A thread local is a variable local to the
Thread, once set they cannot be removed unless overwritten by a different value. While this makes sense, when in the
context of a ThreadPool, this means that all tasks using the Thread also get access to the locals.

And last, but not least, due to the fact it's an OS thread, the cancellation program is rather complex and expensive. To
elaborate on this, if a thread calls Thread#interrupt on another thread, this thread sets the status to Interrupted, and
throws a InterruptedException to all blocking methods. After which the status gets cleared.

The status gets cleared for 2 reasons. First, this allows ThreadPool's to re-use the Thread. Second of all, one
might require some of the Thread's blocking methods after an error.

While the purpose of this behaviour, makes sense. It's just rather error-prone, preferably we'd want to create a new
Thread instead.

To summarize this
- Thread's expose thread locals to everything running in the Thread
- Complex cancellation program
- No concurrency

*Java*

```java
public class Application {
    public static void main(String[] args) {
        new Thread(() -> {
            var server = randomServer();
            var member = server.getRandomMember();

            if (member.hasAcceptedRules()) {
                var action = member.giveRole(Role.STARTER);

                if (action.hadSuccess()) {
                    user.sendMessage("Gave you the starter role!");
                } else {
                    user.sendMessage("Something went wrong, please contact a developer)");
                }
            } else {
                user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
            }
        });
    }
}

```

But at least, the code is readable, no await, no playing around with Task<T>. This runs in parallel, the
concurrency is still being handled by the OS, ineffectively.

It'd be awesome if we somehow, mixed those together in an effective manner? Which is where reactive comes in!


## Solution 3, (Functional) reactive programming

Reactive combines the best and worst of both, it results in a concurrent and parallel method of programming.

To take a look at the definition according to Wikipedia
> Functional reactive programming (FRP) is a programming paradigm for reactive programming (asynchronous dataflow programming) using the building blocks of functional programming (e.g. map, reduce, filter).

As you might notice, this sounds advanced. And unfortunately I'll have to tell you, reactive programming has a
rather steep learning curve.

So to elaborate on what reactive programming is, I'll show you a simple example which we will break down together.

```java
public class Application {
    public static void main(String[] args) {
        Stream.of("yeppers", "something", "buzz", "fuzz", "bar")
                .filter(s -> s.startsWith("b"))
                .map(String::toUpperCase)
                .sorted()
                .forEach(System.out::println);
    }
}
```

A Stream is a so called data-stream. It depends on the kind of data-source how the stream behaves, in this case the
stream instantly consists of all possible elements, and no new ones will be added later. And due to the size of the
stream, there's no need for making it parallel. In this specific example, there's nothing concurrent (like streams
combining) either. This is just, to show how a stream looks.

By using method references, the code becomes a bit more readable, and by (flat)mapping, you don't need the constant
callbacks in callbacks. Sometimes, you will have to wrap your lambda into a method to make it more readable, in the
example below you'll understand why.

```java
public class Application {
    public static void main(String[] args) {
        randomServer()
                .flatmap(Server::getMember)
                .flatmap((member) -> {
                    if (member.hasAcceptedRules()) {
                       return member.giveRole(Role.STARTER)
                               .flatmap(Action::hadSuccess, action -> {
                                  user.sendMessage("Gave you the starter role!");
                               }).onErrorFlatmap((action) -> {
                                  return user.sendMessage("Something went wrong, please contact a developer)");
                               });
                    } else {
                        return user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
                    }
                });
    }
}

```

If you're new to reactive programming, I'm sorry for the heart attack this might has given you. So as you see, concurrent,
parallel, no callback hell, or well it's supposed to be no callback hell.

This looks awful, which is where something important in the programming world comes back. Methods, splitting logic into methods.
If Member#hasAcceptedRules is true, it should instead call a method that contains all the logic, this makes it a lot more readable.
While by far, not even close to perfect. It's still, an improvement. See the result below.

```java
public class Application {
    public static void main(String[] args) {
        randomServer()
                .flatmap(Server::getMember)
                .flatmap((member) -> {
                    if (member.hasAcceptedRules()) {
                        return handleMemberRuleAccept(member);
                    } else {
                        return user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
                    }
                });
    }

    private static RestAction handleMemberRuleAccept(Member member) {
        return member.giveRole(Role.STARTER)
                .flatmap(Action::hadSuccess, action -> {
                    user.sendMessage("Gave you the starter role!");
                }).onErrorFlatmap((action) -> {
                    return user.sendMessage("Something went wrong, please contact a developer)");
                });
    }
}
```

This still isn't something I'd show a beginner, someone needs decent/good programming knowledge to read this, and especially write this.
But before I keep complaining about the steep learning-curve, let's go over the benefits.

- Applies concurrency and parallelism
- A lot of operators making your code more readable (there's method for almost everything possible)
- Alright to read.

And the issues

- Extremely steep learning curve
- Hard to debug

So, is it really worth the hassle? I mean yeah it's performant, and readable for someone with knowledge, 
but it's steep learning-curve makes it something beginners would be scared of. 

## Solution 4, the final boss, green threads

Green threads, fibers, lightweight threads, virtual threads, it doesn't matter how you call them.
They are the solution, in my eyes. You can code like it's sync, while the code runs in concurrency and parallelism.

Let's go over all the issues we found with all the other solutions, and check or these apply here.

### Issues of solution 1, async keyword

- You can only call async methods in async methods, you can't call an async method in a sync method. (unless using a callback)
- Sync functions return a value, async one's return a wrapper around the value (Future, Promise)
- You can't use try-catch, since the async one happens on another thread. (Often in callbacks)
- When using callbacks from code that you never wrote, you have to trust them that the callback actually runs.

All issues, only apply to async methods. So you could say, fibers fixed them by removing async methods all together.


### Issues of solution 2, threads

- Thread's expose thread locals to everything running in the Thread

This issue existed because you need to re-use threads due to the fact that Threads are rather expensive.
But fibers, they aren't expensive, you **should** create a new virtual thread for each task.
So it doesn't expose anything

- Complex cancellation program

Since fibers don't have to be re-used, the complexity is removed. One can just create a new virtual thread if someone goes wrong.

- No concurrency

Fibers are handled concurrently, so don't worry about blocking calls! 

### Issue of solution 3, (Functional) reactive programming

- Extremely steep learning curve

Concurrent might be a little weird to understand at the start, but it doesn't have a steep learning curve. 
You don't need to know what it does, all you need to know is that you shouldn't be afraid to create a thread.

- Hard to read and debug

Since it's sync code, it's easy to read.
Since the same (virtual) thread always runs the code, it's also so much easier to debug.

### Example

So let's, just take a look at an example. Is it actually so perfect, I mean there have to be some issues right?

*Note, this is non-existent Java code, creation of the virtual thread would be a bit different, but everything else would be the same*
```java
public class Application {
   public static void main(String[] args) {
      new VirtualThread(() -> {
         var server = randomServer();
         var member = server.getRandomMember();

         if (member.hasAcceptedRules()) {
            var action = member.giveRole(Role.STARTER);

            if (action.hadSuccess()) {
               user.sendMessage("Gave you the starter role!");
            } else {
               user.sendMessage("Something went wrong, please contact a developer)");
            }
         } else {
            user.sendMessage("You haven't accepted the rules yet, therefor I'll not give you the starter role.");
         }
      });
   }
}
```

There, it looks exactly the same as the second solution, Threads. Difference is that it's concurrent,
and doesn't have the thread local issues, nor complex cancellation program.

More performant you cannot get, 

Alright, in all honesty, there's 1 issue. Some people don't like the fact that the blocking calls are implicit concurrency.
But luckily, this is subjective, I personally don't mind this, since it doesn't require major refactoring like Reactive programming or the async-await keyword.