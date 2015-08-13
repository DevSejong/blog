# 자바8 Concurrency 튜토리얼: Thread/Executor

Welcome to the first part of my Java 8 Concurrency tutorial. This guide teaches you concurrent programming in Java 8 with easily understood code examples. It's the first part out of a series of tutorials covering the Java Concurrency API. In the next 15 min you learn how to execute code in parallel via threads, tasks and executor services.

이 가이드는 코드를 통해서 자바 8의 [concurrent(이하 병렬) 프로그래밍](https://en.wikipedia.org/wiki/Concurrent_computing)을 쉽게 이해하는 것을 목표로 만들어졌습니다. 자바의 [Concurrency API](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html)를 설명하는 튜토리얼 중 첫번째 파트이며, 앞으로 15분 정도의 분량으로 thread, task, executor service를 사용하여 비동기를 어떻게 처리하는지에 대해서 이야기 할 것 입니다.

***

- Part 1: Threads and Executors
- Part 2: Synchronization and Locks
- Part 3: Atomic Variables and ConcurrentMap


- 파트 1: Thread/Executor
- 파트 2: Synchronization/Lock
- 파트 3: Atomic Variables/ConcurrentMap

***

The Concurrency API was first introduced with the release of Java 5 and then progressively enhanced with every new Java release. The majority of concepts shown in this article also work in older versions of Java. However my code samples focus on Java 8 and make heavy use of lambda expressions and other new features. If you're not yet familiar with lambdas I recommend reading my Java 8 Tutorial first.

Concurrency API는 자바5에서 처음으로 소개되었으며, 새로운 자바 버전이 발표될 때마다 조금씩 보강되어 왔습니다. 아래의 가이드에 나오는 이러한 주요 개념들은 오래된 자바에서도 동일하게 사용할 수 있으나, 이 가이드는 자바8을 기준으로 설명하였습니다. 람다 표현식과 같이 자바8에 소개된 새로운 문법을 많이 사용하였으므로 문법에 친숙하지 않을 경우 우선 람다에 대한 [튜토리얼](http://winterbe.com/posts/2014/03/16/java-8-tutorial/) 먼저 참조하시기 바랍니다.

***

## Threads and Runnables

All modern operating systems support concurrency both via processes and threads. Processes are instances of programs which typically run independent to each other, e.g. if you start a java program the operating system spawns a new process which runs in parallel to other programs. Inside those processes we can utilize threads to execute code concurrently, so we can make the most out of the available cores of the CPU.

## Thread와 Runnable

우리가 사용하는 모든 현대적인 OS들은 [프로세스](https://ko.wikipedia.org/wiki/프로세스)와 [스레드](https://ko.wikipedia.org/wiki/스레드)를 사용한 병렬 처리를 지원합니다. 프로그램의 인스턴스인 프로세스는 일반적으로 서로 독립된 상태로 동작합니다. 예를 들어 자바 프로그램을 실행할 경우 OS는 새로운 프로세스를 생성하며, 이 프로세스는 다른 프로그램과 병렬으로 실행됩니다. 이 프로세스들 내부에서 동시에 코드를 실행할 수 있는 스레드를 사용할 수 있으며. CPU에서 가용 코어를 모두 사용할 수 있게 됩니다.

***

Java supports Threads since JDK 1.0. Before starting a new thread you have to specify the code to be executed by this thread, often called the task. This is done by implementing Runnable - a functional interface defining a single void no-args method run() as demonstrated in the following example:

자바는 JDK 1.0 부터 [스레드](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html)를 지원해왔습니다. 새로운 스레드를 시작하기 전에 스레드 내부에서 실행되는 코드(이를 task라고 칭함)를 만들어야 합니다. 이 task는 `Runnable`을 상속함으로써 만들 수 있습니다. `Runnable`은 아래와 같이 반환값과 아규먼트가 없는 매서드 `run()`을 가지고 있는 Functional Interface입니다.


	Runnable task = () -> {
	    String threadName = Thread.currentThread().getName();
	    System.out.println("Hello " + threadName);
	};

	task.run();

	Thread thread = new Thread(task);
	thread.start();

	System.out.println("Done!");


Since Runnable is a functional interface we can utilize Java 8 lambda expressions to print the current threads name to the console. First we execute the runnable directly on the main thread before starting a new thread.

위의 코드에서는 `Runnable` 인터페이스에서 람다 표현식을 사용하여 현재 스레드 이름을 콘솔에 출력하였습니다. 우선 메인 스레드에서 새로운 스레드를 실행하기 전에 `Runnable`을 바로 실행하였습니다.

***

콘솔에서 출력되는 결과는 다음과 같을 것입니다.

	Hello main
	Hello Thread-0
	Done!

Or that:

또는 아래처럼 나올 수 도 있습니다.

***

	Hello main
	Done!
	Hello Thread-0

Due to concurrent execution we cannot predict if the runnable will be invoked before or after printing 'done'. The order is non-deterministic, thus making concurrent programming a complex task in larger applications.

위의 코드는 병렬으로 실행되므로, 마지막에 위치하는 "done"이 Runnable을 활용한 스레드보다 먼저 출력될지 나중에 출력될지를 알 수 없습니다. 정렬순서는 결정할 수 없으므로, 병렬프로그래밍은 커다란 애플리케이션을 만들경우 복잡한 작업입니다.

***

Threads can be put to sleep for a certain duration. This is quite handy to simulate long running tasks in the subsequent code samples of this article:

스레드에서는 특정한 시간동안 스레드를 멈출 수 있는 sleep을 둘 수 있습니다. 다음과 같이 긴 작업을 시뮬레이션 할 경우에 sleep을 편리하게 사용할 수 있습니다.

***

 	Runnable runnable = () -> {
	    try {
	        String name = Thread.currentThread().getName();
	        System.out.println("Foo " + name);
	        TimeUnit.SECONDS.sleep(1);
	        System.out.println("Bar " + name);
	    }
	    catch (InterruptedException e) {
	        e.printStackTrace();
	    }
	};

	Thread thread = new Thread(runnable);
	thread.start();

When you run the above code you'll notice the one second delay between the first and the second print statement. TimeUnit is a useful enum for working with units of time. Alternatively you can achieve the same by calling Thread.sleep(1000).

위의 코드를 실행할 경우, 첫번째와 두번째 출력사이에 1초간 딜레이가 발생하는 것을 확인할 수 있을 것 입니다. `TimeUnit`은 시간의 단위를 기준으로 작업을 진행할 수 있도록 도와주는 enum입니다. `Thread.sleep(1000)`과 동일한 역할을 합니다.

*** 

Working with the Thread class can be very tedious and error-prone. Due to that reason the **Concurrency API** has been introduced back in 2004 with the release of Java 5. The API is located in package `java.util.concurrent` and contains many useful classes for handling concurrent programming. Since that time the Concurrency API has been enhanced with every new Java release and even Java 8 provides new classes and methods for dealing with concurrency.

스레드 클래스를 사용하는 방식은 매우 지루하며, 에러가 나기 쉬운 구조입니다. 때문에 2004년 후반 자바 5가 릴리즈에서 **Concurrency API**가 소개되었습니다. 이 API들은 `java.util.concurrent`패키지에서 찾을 수 있으며, 병렬 프로그램을 다룰 수 있는 유용한 클래스가 포함되어 있습니다. 새로운 자바가 발표될 때 마다 이 API는 계속 보강되어 왔으며, 자바 8에서는 병렬 프로그램을 다룰 수 있는 새로운 클래스와 메서드가 추가되었습니다.

***

Now let's take a deeper look at one of the most important parts of the Concurrency API - the executor services.

아래에서는 이 Concurrency API의 가장 중요한 부분중 하나인 Executor서비스에 대해서 자세히 알아보도록 합니다.

***

## Executors

The Concurrency API introduces the concept of an `ExecutorService` as a higher level replacement for working with threads directly. Executors are capable of running asynchronous tasks and typically manage a pool of threads, so we don't have to create new threads manually. All threads of the internal pool will be reused under the hood for revenant tasks, so we can run as many concurrent tasks as we want throughout the life-cycle of our application with a single executor service.

## Executor

Concurrency API에서는 직접 스레드를 사용할 수 있는 높은 레벨의 `ExecutorService`가 포함되어 있습니다. Executor는 비동기 작업을 실행할 수 있고, 스레드풀을 관리할 수 있는 기능을 제공합니다. 때문에 직접 스레드를 새롭게 생성하지 않아도 됩니다. 내부의 풀에 소속된 모든 스레드들은 재사용될 수 있으므로, 우리는 애플리케이션 생명주기 내에서 하나의 Executor service를 사용하여 많은 동행 작업을 처리할 수 있습니다.

*** 

This is how the first thread-example looks like using executors:

아래는 executor를 사용한 예제입니다.

***

    ExecutorService executor = Executors.newSingleThreadExecutor();
    executor.submit(() -> {
        String threadName = Thread.currentThread().getName();
        System.out.println("Hello " + threadName);
    });

    // => Hello pool-1-thread-1

The class Executors provides convenient factory methods for creating different kinds of executor services. In this sample we use an executor with a thread pool of size one.

Excutor 클래스는 다양한 형태의 Executor Service를 만들 수 있도록 팩토리 메서드를 제공합니다. 위 예제에서는 하나의 스레드풀을 사용하도록 지정하였습니다.

***

The result looks similar to the above sample but when running the code you'll notice an important difference: the java process never stops! Executors have to be stopped explicitly - otherwise they keep listening for new tasks.

결과는 위의 스레드를 사용한 코드와 동일한 것 처럼 보이지만 중요한 차이점이 있습니다. 자바 프로세스가 절대 종료되지 않는다는 것입니다. Executor는 명시적으로 종료되지 않습니다. (새로운 작업을 받을 수 있는 대기 상태로 유지됩니다.)

***

An `ExecutorService` provides two methods for that purpose: `shutdown()` waits for currently running tasks to finish while `shutdownNow()` interrupts all running tasks and shut the executor down immediately.

`ExecutorService`는 종료를 위해서 두가지 메서드를 제공합니다. `shutdown()`은 현재 진행중인 작업들이 끝나는것을 기다린 뒤, `shutdownNow()`는 모든 작업을 한꺼번에 멈춘뒤 Executor를 종료합니다.

***

This is the preferred way how I typically shutdown executors:

아래는 Executor를 종료하는 일반적인 방법입니다.

***

	try {
	    System.out.println("attempt to shutdown executor");
	    executor.shutdown();
	    executor.awaitTermination(5, TimeUnit.SECONDS);
	}
	catch (InterruptedException e) {
	    System.err.println("tasks interrupted");
	}
	finally {
	    if (!executor.isTerminated()) {
	        System.err.println("cancel non-finished tasks");
	    }
	    executor.shutdownNow();
	    System.out.println("shutdown finished");
	}

The executor shuts down softly by waiting a certain amount of time for termination of currently running tasks. After a maximum of five seconds the executor finally shuts down by interrupting all running tasks.

Executor는 먼저 종료신호를 보내며, 지정된 시간만큼 현재 진행작업이 끝나도록 대기합니다. 지정한 시간 5초가 지나면, Executor는 모든 실행중 태스크에 interrupt예외를 발생시켜 강제종료시키며 최종적으로 Executor를 종료합니다.

***

## Callables and Futures

In addition to `Runnable` executors support another kind of task named `Callable`. Callables are functional interfaces just like runnables but instead of being `void` they return a value.

This lambda expression defines a callable returning an integer after sleeping for one second:

## Callable / Futures

Executor는 `Runnable`에 더해 `Callable`을 사용하여 task를 만들 수 있습니다. `Callable`은 `Runnalbe`과 마찬가지로 functional interface이지만, 반환값을 가지고 있다는 차이점이 있습니다.

아래의 예제에서는 람다표현식을 통해서, 1초간 정지한 다음 Integer값을 반환하는 예제입니다.

***

	Callable<Integer> task = () -> {
	    try {
	        TimeUnit.SECONDS.sleep(1);
	        return 123;
	    }
	    catch (InterruptedException e) {
	        throw new IllegalStateException("task interrupted", e);
	    }
	};

Callables can be submitted to executor services just like runnables. But what about the callables result? Since `submit()` doesn't wait until the task completes, the executor service cannot return the result of the callable directly. Instead the executor returns a special result of type `Future` which can be used to retrieve the actual result at a later point in time.

`Callable`은  `Runnalbe`과 마찬가지로 `submit()`을 사용하여 작업을 실행할 수 있습니다. 그렇다면 실행 결과는 어떻게 받아올 수 있을까요? `submit()`은 작업이 완료되는것을 기다리지 않으므로, ExecutorService는 Callable을 실행한 결과를 바로 가져올 수 없습니다. 대신에 `Future`라는 특별한 타입의 값을 반환합니다. `Future`는 실행 결과를 원하는 시간에 가져올 수 있도록 제공합니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(1);
	Future<Integer> future = executor.submit(task);

	System.out.println("future done? " + future.isDone());

	Integer result = future.get();

	System.out.println("future done? " + future.isDone());
	System.out.print("result: " + result);

After submitting the callable to the executor we first check if the future has already been finished execution via `isDone()`. I'm pretty sure this isn't the case since the above callable sleeps for one second before returning the integer.

`submit()`을 사용하여 `Callable`을 실행시킨 경우, 작업이 종료되었는지 여부는 `isDone()`메서드를 사용하여 확인할 수 있습니다. 이전 코드에서 우리가 만든 `Callable`에서는 1초간 sleep을 걸어놓았기에, 처음의 `isDone()`이 호출되는 시점에서는 작업이 진행중인 상태일 것이라고 예상 할 수 있을 것 입니다.

***

Calling the method `get()` blocks the current thread and waits until the callable completes before returning the actual result `123`. Now the future is finally done and we see the following result on the console:

메서드 `get()`을 호출할 경우, `Callable`의 결과인 `123`이 나올 때 까지 스레드를 정지합니다. 이때서야 비로소 작업이 끝나게되고, 아래와 같은 결과를 콘솔에서 확인할 수 있습니다.

***

	future done? false
	future done? true
	result: 123

Futures are tightly coupled to the underlying executor service. Keep in mind that every non-terminated future will throw exceptions if you shutdown the executor:

`Future`는 `ExecutorService`와 타이트하게 결합되어 있습니다. 때문에 `Executor`를 종료한 경우, 작업이 종료되지 않은 `Future`들은 에러를 발생시킨다는 사실을 기억하셔야 합니다.

***

	executor.shutdownNow();
	future.get();

You might have noticed that the creation of the executor slightly differs from the previous example. We use `newFixedThreadPool(1)` to create an executor service backed by a thread-pool of size one. This is equivalent to `newSingleThreadExecutor()` but we could later increase the pool size by simply passing a value larger than one.

위의 예제는 이전의 예제와 살짝 다른 부분이 있습니다. `newFixedThreadPool(1)`을 사용하여 `ExecutorService`의 스레드 풀을 하나만 가지게끔 지정하였습니다. 이전 예제의 `newSingleThreadExecutor()`와 동일한 역할을 하고 있습니다. 파라미터의 숫자를 늘림으로써 스레드풀의 숫자를 더 늘릴수 있을 것 입니다.

***

## Timeouts

Any call to `future.get()` will block and wait until the underlying callable has been terminated. In the worst case a callable runs forever - thus making your application unresponsive. You can simply counteract those scenarios by passing a timeout:

## Timeout

`future.get()`을 호출할 경우 현재 스레드는 callable을 구현한 작업이 끝날때 까지 정지한 상태로 대기합니다. callable의 작업이 종료되지 않고 계속 실행되는 경우, 애플리케이션이 동작하지 않게 될 수도 있습니다. 이러한 경우 아래의 예제와 같이 timeout을 지정하여 대비할 수 있습니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(1);

	Future<Integer> future = executor.submit(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
        return 123;
    }
    catch (InterruptedException e) {
        throw new IllegalStateException("task interrupted", e);
    }
	});

	future.get(1, TimeUnit.SECONDS);

Executing the above code results in a TimeoutException:

위의 코드를 실행할 경우 아래와 같이 `TimeoutException`이 발생하게 됩니다.

***

	Exception in thread "main" java.util.concurrent.TimeoutException
    at java.util.concurrent.FutureTask.get(FutureTask.java:205)

You might already have guessed why this exception is thrown: We specified a maximum wait time of one second but the callable actually needs two seconds before returning the result.

위의 예외가 왜 발생하는지에 대해서 벌써 알고 계실꺼라고 생각합니다. 위의 코드에서 최대 대기시간을 1초로 설정하였지만, 실제 결과가 나오기까지는 2초가 걸리기 때문입니다.

***

## InvokeAll

Executors support batch submitting of multiple callables at once via invokeAll(). This method accepts a collection of callables and returns a list of futures.

## invokeAll

`Executor`는 여러개의 `Callable`을 한번에 실행할 수 있는 `invokeAll()`을 지원합니다. 이 메서드는 `Callable`의 컬렉션을 받을 수 있으며 리스트의 형식으로 `Future`타입의 값을 반환합니다.

***

	ExecutorService executor = Executors.newWorkStealingPool();

	List<Callable<String>> callables = Arrays.asList(
        () -> "task1",
        () -> "task2",
        () -> "task3");

	executor.invokeAll(callables)
    .stream()
    .map(future -> {
        try {
            return future.get();
        }
        catch (Exception e) {
            throw new IllegalStateException(e);
        }
    })
    .forEach(System.out::println);

In this example we utilize Java 8 functional streams in order to process all futures returned by the invocation of `invokeAll`. We first map each future to its return value and then print each value to the console. If you're not yet familiar with streams read my [Java 8 Stream Tutorial](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/).

위 예제에서는 자바8의 stream을 활용하여 `invokeAll`의 모든 결과의 반환값이 나올때 까지 실행하도록 하였습니다. 각각의 future들의 리턴값을 콘솔에 출력하도록 매핑하였습니다. 자바8의 stream문법에 익숙하지 않은 경우 [자바8 스트림 튜토리얼](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/)을 참조하시기 바랍니다.

***

## InvokeAny

Another way of batch-submitting callables is the method `invokeAny()` which works slightly different to `invokeAll()`. Instead of returning future objects this method blocks until the first callable terminates and returns the result of that callable.

`Callable`을 실행할 수 있는 다른 방법은 `invokeAny()`를 사용하는 것입니다. `Future`타입의 컬렉션이 반환되는 `invokeAll()`과는 달리 이 메서드는 작업 중 하나의 값이 완료될 때 까지 기다린 뒤 결과를 반환하고, 다른 작업을 종료시킵니다.

***

In order to test this behavior we use this helper method to simulate callables with different durations. The method returns a callable that sleeps for a certain amount of time until returning the given result:

위의 동작을 테스트해보기 위해서 callable의 실행시간을 다르게 설정해보도록 하겠습니다. 아래는 callable을 호출할 경우 지정된 시간 동안 멈춘 뒤 결과를 반환하도록 작성된 코드입니다.

***

	Callable<String> callable(String result, long sleepSeconds) {
    return () -> {
        TimeUnit.SECONDS.sleep(sleepSeconds);
        return result;
    };
}

We use this method to create a bunch of callables with different durations from one to three seconds. Submitting those callables to an executor via `invokeAny()` returns the string result of the fastest callable - in that case task2:

위에서 만든 callable을 활용하여 1초부터 3초까지 각기 다른 시간이 걸리도록 설정하였습니다. `Executor`에서 `invokeAny()`를 통해서 실행될 경우 가장 빠른 실행결과가 나오게 됩니다. 아래의 경우에서는 1초를 설정한 `task2`가 출력됩니다.

***

	ExecutorService executor = Executors.newWorkStealingPool();

	List<Callable<String>> callables = Arrays.asList(
    callable("task1", 2),
    callable("task2", 1),
    callable("task3", 3));

	String result = executor.invokeAny(callables);
	System.out.println(result);

	// => task2

The above example uses yet another type of executor created via `newWorkStealingPool()`. This factory method is part of Java 8 and returns an executor of type `ForkJoinPool` which works slightly different than normal executors. Instead of using a fixed size thread-pool ForkJoinPools are created for a given parallelism size which per default is the number of available cores of the hosts CPU.

위의 예제에서는 `newWorkStealingPool()`를 사용하여 `Executor`를 생성하고 있습니다. 이 팩터리 메서드는 자바8에서부터 지원되며, `ForkJoinPool`을 Executor 타입으로 반환합니다. 기존의 고정된 스레드풀을 지정하는 것과는 다르게 [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)은 프로그램을 실행하는 CPU 코어의 숫자를 기준으로 스레드 풀을 생성합니다.

***

ForkJoinPools exist since Java 7 and will be covered in detail in a later tutorial of this series. Let's finish this tutorial by taking a deeper look at scheduled executors.

`ForkJoinPool`은 자바7에서 추가되었습니다. 더욱 자세한 이야기는 이어지는 튜토리얼에서 보다 자세하게 설명하도록 하겠습니다. 이제 이 튜토리얼의 마지막인 `Scheduled Executor`에 대해서 알아보도록 합시다. 

***

## Scheduled Executors

We've already learned how to submit and run tasks once on an executor. In order to periodically run common tasks multiple times, we can utilize scheduled thread pools.

## ScheduledExecutor

지금까지는 `Executor`를 활용하여 일회성 작업들을 어떻게 만들고 실행하는지에 대해서 이야기했습니다. 만약 주기적으로 여러번 실행되어야 하는 작업이 필요하다면  scheduled 스레드풀을 사용하여 구현할 수 있습니다.

***

A `ScheduledExecutorService` is capable of scheduling tasks to run either periodically or once after a certain amount of time has elapsed.

`ScheduledExecutorService`을 사용하면, 주기적으로 또는 작업이 끝난뒤 일정시간 후 다시 작업을 하는 등의 스케쥴링 설정이 가능합니다.

***

This code sample schedules a task to run after an initial delay of three seconds has passed:

아래의 예제에서는 초기의 딜레이 3초 뒤 작업이 진행됩니다.

***

	ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

	Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());
	ScheduledFuture<?> future = executor.schedule(task, 3, TimeUnit.SECONDS);

	TimeUnit.MILLISECONDS.sleep(1337);

	long remainingDelay = future.getDelay(TimeUnit.MILLISECONDS);
	System.out.printf("Remaining Delay: %sms", remainingDelay);

Scheduling a task produces a specialized future of type `ScheduledFuture` which - in addition to `Future` - provides the method `getDelay()` to retrieve the remaining delay. After this delay has elapsed the task will be executed concurrently.

스케쥴링 작업의 결과로는 조금은 독특한 `Future` 타입인 `ScheduledFuture`가 생성됩니다.  이 타입에서는 `getDelay()`를 호출하여 얼마만큼의 딜레이가 남았는지 확인할 수 있습니다. 이 딜레이가 완료된 후에 작업이 시작되게 됩니다.

***

In order to schedule tasks to be executed periodically, executors provide the two methods `scheduleAtFixedRate()` and `scheduleWithFixedDelay()`. The first method is capable of executing tasks with a fixed time rate, e.g. once every second as demonstrated in this example:

주기적으로 실행되는 스케쥴 작업을 위해서 `Executor`에서는 두개의 메서드 `scheduleAtFixedRate()`과 `scheduleWithFixedDelay()`를 제공합니다. 첫번째 메서드는 고정된 시간을 기준으로 작업을 진행합니다. 아래의 코드에서는 매 초마다 현재시간이 콘솔에 출력됩니다.

***

	ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

	Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());

	int initialDelay = 0;
	int period = 1;

	executor.scheduleAtFixedRate(task, initialDelay, period, TimeUnit.SECONDS);

Additionally this method accepts an initial delay which describes the leading wait time before the task will be executed for the first time.

추가적으로 이 메서드는 작업이 시작되기 전에 초기 딜레이 값을 지정할 수 있습니다. 

***

Please keep in mind that `scheduleAtFixedRate()` doesn't take into account the actual duration of the task. So if you specify a period of one second but the task needs 2 seconds to be executed then the thread pool will working to capacity very soon.

기억해야 하는 사실은 `scheduleAtFixedRate()`는 실제 작업이 얼마만큼 걸리는지에 대해서 고려하지 않는다는 것입니다. 만약 반복 주기를 1초라고 지정하였지만, 실제 작업이 2초가 걸린다면 스레드풀은 곧 차버리게 될 것입니다.

***

In that case you should consider using `scheduleWithFixedDelay()` instead. This method works just like the counterpart described above. The difference is that the wait time period applies between the end of a task and the start of the next task. For example:

이러한 경우라면 `scheduleWithFixedDelay()`를 사용하는 것을 고려해 보아야 합니다. 이 메서드는 위의 상황에서 대체제가 될 것 입니다. `scheduleAtFixedRate()`와 다른점은 작업이 종료된 후 부터 지정된 시간을 기다린 후 다음 작업을 시작한다는 점 입니다.

***

	ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

	Runnable task = () -> {
    try {
        TimeUnit.SECONDS.sleep(2);
        System.out.println("Scheduling: " + System.nanoTime());
    }
    catch (InterruptedException e) {
        System.err.println("task interrupted");
    }
};

	executor.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);

This example schedules a task with a fixed delay of one second between the end of an execution and the start of the next execution. The initial delay is zero and the tasks duration is two seconds. So we end up with an execution interval of 0s, 3s, 6s, 9s and so on. As you can see `scheduleWithFixedDelay()` is handy if you cannot predict the duration of the scheduled tasks.

위 예제에서는 작업이 종료된 후 다음 작업이 시작되기 전까지 딜레이를 1초로 지정하였습니다. 초기 딜레이는 0이며, 작업은 2초가 걸립니다. 그러므로 위 예제에서는 3초를 주기로 작업이 진행되는 것을 확인할 수 있을 것 입니다. 이처럼 `scheduleWithFixedDelay()`는 작업이 얼마만큼 걸릴지 모르는 상황에서 유용하게 사용할 수 있습니다.

***

This was the first part out of a series of concurrency tutorials. I recommend practicing the shown code samples by your own. You find all code samples from this article on GitHub, so feel free to fork the repo and give me star.

이상으로 concurrency 튜토리얼의 첫번째 파트를 마칩니다. 위의 코드를 스스로 실행해보시는 것을 강력하게 권장합니다. 또한 예제에 대한 샘플은 [Github](https://github.com/winterbe/java8-tutorial)에서 찾을 수 있습니다. fork를 하고 star를 주는 것에 대해서 주저하지 마세요.

***

