Concurrency & Rate Limiting
===========================

.. note:: The Hangfire.Throttling package is a part of `Hangfire.Ace <https://www.hangfire.io/ace/>`_ extensibility set and available on the private NuGet feed.

The Hangfire.Throttling package contains advanced types and methods to apply concurrency and rate limits directly to our background jobs without touching any logic related to queues, workers, servers or using additional services. So we can control how many particular background jobs are running at the same point in time or within a specific time window.

Throttling is performed asynchronously by rescheduling jobs to a later time or deleting them when throttling condition is met, depending on the configured behavior. And while throttled jobs are waiting for their turn, our workers are free to process other enqueued background jobs.

The primary focus of this package is to provide a simpler way of reducing the load on external resources. Databases or third-party services affected by background jobs may suffer from additional concurrency, causing increased latency and error rates. While the standard solution of using different queues with a constrained number of workers does work well enough, it requires additional infrastructure planning and may lead to under utilization. While in contrast, throttling primitives are much easier to use for this purpose.

.. admonition:: Everything works on a best-effort basis
   :class: warning

   While it can be possible to use this package to enforce proper synchronization and concurrency control over background jobs, it is very hard to achieve it due to the complexity of distributed processing. There are a lot of things to consider, including appropriate storage configuration, and a single mistake will ruin everything.

   Throttlers apply only to different background jobs, and there is no reliable way to prevent multiple executions of the same background job other than by using transactions within the background job method itself. ``DisableConcurrentExecution`` may help a bit by narrowing the safety violation surface, but it heavily relies on an active connection, which may be broken (and lock is released) without any notification for our background job.

Hangfire.Throttling provides the following primitives, all of them are implemented as regular state changing filters that run when a worker is starting or completing a background job. They form two groups, depending on their acquire and release behavior.

`Concurrency Limiters`_

* `Mutexes`_ -- allow only a single background job to be running concurrently.
* `Semaphores`_ – limit how many background jobs are allowed to run concurrently.

`Rate Limiters`_

* `Fixed Window Counters`_ – limit how many job executions are allowed within a given fixed time interval.
* `Sliding Window Counters`_ – limit how many job executions are allowed to run within any specific time interval.
* `Dynamic Window Counters`_ – allow the creation of window counters dynamically depending on job arguments.

Requirements
------------

Supported only for :doc:`Hangfire.SqlServer <../configuration/using-sql-server>` (better to use ≥ 1.7) and :doc:`Hangfire.Pro.Redis <../configuration/using-redis>` (recommended ≥ 2.4.0) storages. Community storage support will be denoted later after defining correctness conditions for that storage.

Installation
------------

The package is available on a private Hangfire.Ace NuGet feed (that's different from Hangfire.Pro one), please see the `Downloads page <https://www.hangfire.io/ace/downloads.html>`_ to learn how to use it. After registering the private feed, we can install the ``Hangfire.Throttling`` package by editing our ``.csproj`` file for new project types:

.. code-block:: xml

   <PackageReference Include="Hangfire.Throttling" Version="1.0.*" />

Alternatively we can use Package Manager Console window to install it using the ``Install-Package`` command as shown below.

.. code-block:: powershell

   Install-Package Hangfire.Throttling

Configuration
-------------

The only configuration method required for throttlers is the ``IGlobalConfiguration.UseThrottling`` extension method. If we do not call this method, every background job decorated with any throttling filter will be eventually moved to the failed state.

.. code-block:: c#
   :emphasize-lines: 3

   GlobalConfiguration.Configuration
       .UseXXXStorage()
       .UseThrottling();

The ``UseThrottling`` method will register all the required filters to make throttling work and adds new pages to the Dashboard UI. We can also configure the default throttling action to tell the library whether to retry or delete a background job when it is throttled, and specify minimal retry delay (should be greater or equal to 15 seconds) useful for `Concurrency Limiters`_.

.. code-block:: c#
   :emphasize-lines: 3

   GlobalConfiguration.Configuration
       .UseXXXStorage()
       .UseThrottling(ThrottlingAction.RetryJob, TimeSpan.FromMinutes(1));

When using custom ``IJobFilterProvider`` instance that is resolved via some kind of IoC container, we can use another available overload of the ``UseThrottling`` method as shown below. It is especially useful for ASP.NET Core applications that are heavily driven by built-in dependency injection.

.. code-block:: c#
   :emphasize-lines: 3

   GlobalConfiguration.Configuration
       .UseXXXStorage()
       .UseThrottling(provider.Resolve<IJobFilterProvider>, ThrottlingAction.RetryJob, TimeSpan.FromMinutes(1));

Usage
-----

Most of the throttling primitives are required to be created first using the ``IThrottlingManager`` interface. Before creating it though, we should pick a unique *Resource Identifier* we can use later to associate particular background jobs with this or that throttler instance.

The Resource Identifier is a generic string with a maximum of 100 characters and is just a reference we need to pick to allow Hangfire to know where to get the primitive's metadata. Resource Identifiers are isolated between different primitive types, but it is better not to use the same identifiers so as not to confuse anyone.

In the following example, a semaphore is created with the ``orders`` identifier and a limit of 20 concurrent background jobs. Please see later sections to learn how to create other throttling primitives. We will use this semaphore after a while.

.. code-block:: c#

   using Hangfire.Throttling;

   IThrottlingManager manager = new ThrottlingManager();
   manager.AddOrUpdateSemaphore("orders", new SemaphoreOptions(limit: 20));

Adding Attributes
~~~~~~~~~~~~~~~~~

Throttlers are regular background job filters and can be applied to a particular job by using corresponding attributes as shown in the following example. After adding these attributes, the state changing pipeline will be modified for all the methods of the defined interface.

.. code-block:: c#

   using Hangfire.Throttling;

   [Semaphore("orders")]
   public interface IOrderProcessingJobsV1
   {
       int CreateOrder();

       [Mutex("orders:{0}")]
       void ProcessOrder(long orderId);

       [Throttling(ThrottlingAction.DeleteJob, minimumDelayInSeconds: 120)]
       void CancelOrder(long orderId);
   }

Throttling
~~~~~~~~~~

Throttling happens when the throttling condition of one of the applied throttlers was not satisfied. It can be configured either globally or locally, and the default throttling action is to schedule the background job to run *one minute* (also can be configured) later. After acquiring a throttler, it is not released until job is moved to a final state to prevent part effects.

Before processing the ``CreateOrder`` method in the example above, a worker will attempt to acquire a semaphore first. On successful acquisition, the background job will be processed immediately. But if the acquisition fails, the background job is throttled. The default throttling action is ``RetryJob``, so it will be moved to the ``ScheduledState`` with a default delay of 1 minute.

For the ``ProcessOrder`` method, the worker will attempt to acquire *both* semaphore and mutex. So if the acquisition of a mutex or semaphore, or both of them fails, the background job will be throttled and retried, releasing the worker. 

And for the ``CancelOrder`` method, the default throttling action is changed to the ``DeleteJob`` value. So when the semaphore cannot be acquired for that job, it will be deleted instead of re-scheduled.

Removing Attributes
~~~~~~~~~~~~~~~~~~~

It is better not to remove the throttling attributes directly when deciding to remove the limits on the particular method, especially for `Concurrency Limiters`_, because some of them may not be released properly. Instead, set the ``Mode`` property to the ``ThrottlerMode.Release`` value (default is ``ThrottlerMode.AcquireAndRelease``) of a corresponding limiter first.

.. code-block:: c#
   :emphasize-lines: 6

   using Hangfire.Throttling;

   [Semaphore("orders")]
   public interface IOrderProcessingJobsV1
   {
       [Mutex("orders:{0}", Mode = ThrottlerMode.Release)]
       Task ProcessOrderAsync(long orderId);

       // ...
   }

In this mode, throttlers will not be applied anymore, only released. So when all the background jobs processed and corresponding limiters were already released, we can safely remove the attribute. `Rate Limiters`_ do not run anything on the Release stage and are expired automatically, so we do not need to change the mode before their removal.

Strict Mode
~~~~~~~~~~~

Since the primary focus of the library is to reduce pressure on other services, throttlers are released by default when background jobs move out of the *Processing* state. So when you retry or re-schedule a running background job, any *Mutexes* or *Semaphores* will be released immediately which lets other jobs acquire them. This mode is called *Relaxed*.

Alternatively you can use *Strict Mode* to release throttlers only when a background job has fully completed, e.g. moved to a final state (such as *Succeeded* or *Deleted*, but not the *Failed* one). This is useful when your background job produces multiple side effects, and you do not want to let other background jobs examine these partial effects.

You can turn on the *Strict Mode* by applying the ``ThrottlingAttribute`` on a method and using its ``StrictMode`` property as shown below. When multiple throttlers are defined, Strict Mode is applied to all of them. Please note it affects only *Concurrency Limiters* and does not affect *Rate Limiters*, since they do not invoke anything when released.

.. code-block:: c#
   :emphasize-lines: 2

   [Mutex("orders:{0}")]
   [Throttling(StrictMode = true)]
   Task ProcessOrderAsync(long orderId);

In either mode, throttler's release and background job's state transition are performed in the same transaction.

Concurrency Limiters
--------------------

Mutexes
~~~~~~~

Mutex prevents concurrent execution of *multiple* background jobs that share the same resource identifier. Unlike other primitives, they are created dynamically so we do not need to use ``IThrottlingManager`` to create them first. All we need is to decorate our background job methods with the ``MutexAttribute`` filter and define what resource identifier should be used.

.. code-block:: csharp

   [Mutex("my-resource")]
   public void MyMethod()
   {
       // ...
   }

When we create multiple background jobs based on this method, they will be executed one after another on a best-effort basis with the limitations described below. If there is a background job protected by a mutex currently executing, other executions will be throttled (rescheduled by default a minute later), allowing a worker to process other jobs without waiting.

.. admonition:: Mutex does not prevent the simultaneous execution of the same background job
   :class: warning

   As there are no reliable automatic failure detectors in distributed systems, it is possible that the same job is being processed on different workers in some corner cases. Unlike OS-based mutexes, mutexes in this package do not protect from this behavior so develop accordingly.

   ``DisableConcurrentExecution`` filter may reduce the probability of the violation of this safety property, but the only way to guarantee it is to use transactions or CAS-based operations in your background jobs to make them idempotent.

   If a background job protected by a mutex is unexpectedly terminated, it will simply re-enter the mutex. It will be held until the background job is moved to the final state (Succeeded, Deleted, but not Failed).

We can also create multiple background job methods that share the same resource identifier, and mutual exclusive behavior will span all of them, regardless of the method name.

.. code-block:: csharp

   [Mutex("my-resource")]
   public void FirstMethod() { /* ... */ }

   [Mutex("my-resource")]
   public void SecondMethod() { /* ... */ }

Since mutexes are created dynamically, it is possible to use a dynamic resource identifier based on background job arguments. To define it, we should use String.Format-like templates, and during invocation all the placeholders will be replaced with actual job arguments. But make sure everything is lower-cased and contains only alphanumeric characters with limited punctuation – no rules except maximum length and case insensitivity is enforced, but it is better to keep identifiers simple and smart.

.. admonition:: The maximum length of resource identifiers is 100 characters
   :class: note

   Please keep this in mind especially when using dynamic resource identifiers.

.. code-block:: csharp

   [Mutex("orders:{0}")]
   public void ProcessOrder(long orderId) { /* ... */ }

   [Mutex("newsletters:{0}:{1}")]
   public void ProcessNewsletter(int tenantId, long newsletterId) { /* ... */ }

Throttling Batches
++++++++++++++++++

By default the background job id is used to identify the current owner of a particular mutex. But since version 1.3 it is possible to use any custom value from a given job parameter. With this feature we can throttle entire batches, since we can pass the ``BatchId`` job parameter that is used to store the batch identifier. To accomplish this, we need to create two empty methods with ``ThrottlerMode.Acquire`` and ``ThrottlerMode.Release`` semantics that will acquire and release a mutex:

.. code-block:: c#

   [Mutex("orders-api", Mode = ThrottlerMode.Acquire, ParameterName = "BatchId")] 
   public static void StartBatch() { /* Doesn't do anything */ }

   [Mutex("orders-api", Mode = ThrottlerMode.Release, ParameterName = "BatchId")] 
   public static void CompleteBatch() { /* Doesn't do anything */ }

And then create a batch as a chain of continuations, starting with the ``StartBatch`` method and ending with the ``CompleteBatch`` method. Please note that the last method is created with the ``BatchContinuationOptions.OnAnyFinishedState`` option to release the throttler even if some of our background jobs completed non-successfully (deleted, for example).

.. code-block:: c#

   BatchJob.StartNew(batch =>
   {
       var startId = batch.Enqueue(() => StartBatch());
       var bodyId = batch.ContinueJobWith(startId, nestedBatch =>
       {
           for (var i = 0; i < 5; i++) nestedBatch.Enqueue(() => Thread.Sleep(5000));
       });

       batch.ContinueBatchWith(
           bodyId,
           () => CompleteBatch(),
           options: BatchContinuationOptions.OnAnyFinishedState);
   });

In this case the batch identifier will be used as an owner, and entire batch will be protected by a mutex, preventing other batches from running simultaneously.

Semaphores
~~~~~~~~~~

Semaphore limits concurrent execution of multiple background jobs to a certain maximum number. Unlike mutexes, semaphores should be created first using the ``IThrottlingManager`` interface with the maximum number of concurrent background jobs allowed. The ``AddOrUpdateSemaphore`` method is idempotent, so we can safely place it in the application initialization logic.

.. code-block:: csharp

   IThrottlingManager manager = new ThrottlingManager();
   manager.AddOrUpdateSemaphore("newsletter", new SemaphoreOptions(maxCount: 100));

We can also call this method on an already existing semaphore, and in this case the maximum number of jobs will be updated. If the background jobs that use this semaphore are currently executing, there may be a temporary violation that will eventually be fixed. So if the number of background jobs is higher than the new ``maxCount`` value, no exception will be thrown, but new background jobs will be unable to acquire a semaphore. And when all of those background jobs finished, ``maxCount`` value will be satisfied.

We should place the ``SemaphoreAttribute`` filter on a background job method and provide a correct resource identifier to link it with an existing semaphore. If the semaphore with the given resource identifier does not exist or was removed, an exception will be thrown at run-time, and the background job will be moved to the Failed state.

.. code-block:: csharp

   [Semaphore("newsletter")]
   public void SendNewsletter() { /* ... */ }

.. admonition:: Multiple executions of the same background job count as 1
   :class: warning

   As with mutexes, multiple invocations of the same background job are not respected and counted as 1. So actually it is possible that more than the given count of background job methods are running concurrently. As before, we can use ``DisableConcurrentExecution`` to reduce the probability of this event, but we should be prepared for this anyway.

As with mutexes, we can apply the ``SemaphoreAttribute`` with the same resource identifier to multiple background job methods, and all of them will respect the behavior of a given semaphore. However dynamic resource identifiers based on arguments are not allowed for semaphores as they are required to be created first.

.. code-block:: csharp

   [Semaphore("newsletter")]
   public void SendMonthlyNewsletter() { /* ... */ }

   [Semaphore("newsletter")]
   public void SendDailyNewsletter() { /* ... */ }

An unused semaphore can be removed in the following way. Please note that if there are any associated background jobs that are still running, an ``InvalidOperationException`` will be thrown (see `Removing Attributes`_ to avoid this scenario). This method is idempotent, and will simply succeed without performing anything when the corresponding semaphore does not exist.

.. code-block:: csharp

   manager.RemoveSemaphoreIfExists("newsletter");

Rate Limiters
--------------

Fixed Window Counters
~~~~~~~~~~~~~~~~~~~~~~

Fixed Window Counters limit the number of *background job executions* allowed to run in a specific fixed time window. The entire time line is divided into static intervals of a predefined length, regardless of the actual job execution times (unlike in `Sliding Window Counters`_). 

The Fixed Window is required to be created first and we can do this in the following way. First, we need to pick some resource identifier unique for our application that will be used later when applying an attribute. Then specify the upper limit as well as the length of an interval (minimum 1 second) via the options.

.. code-block:: csharp

   IThrottlingManager manager = new ThrottlingManager();
   manager.AddOrUpdateFixedWindow("github", new FixedWindowOptions(5000, TimeSpan.FromHours(1)));

After creating a fixed window, simply apply the ``FixedWindowAttribute`` filter on one or multiple background job methods, and their state changing pipeline will be modified to apply the throttling rules.

.. code-block:: csharp

   [FixedWindow("github")]
   public void ProcessCommits() { /* ... */ }

When a background job associated with a Fixed Window is about to execute, the current time interval is queried to see the number of already performed job executions. If it is less than the limit value, then the background job is executed. If not, the background job is throttled (scheduled for the next interval by default).

When it is time to stop using the Fixed Window, we should remove all the corresponding ``FixedWindowAttribute`` filters first from our jobs, and then simply call the following method. There is no need to use the ``Release`` mode for Fixed Windows as in `Concurrency Limiters`_, because they do not do anything during this phase.

.. code-block:: csharp

   manager.RemoveFixedWindowIfExists("github");

The Fixed Window Counter is a special case of the Sliding Window Counter described in the next section, with a single bucket. It does not enforce the limitation that *for any given time interval there will be no more than X executions*. So it is possible for one-hour length interval with maximum 4 executions to have 4 executions at 12:59 and another 4 just in a minute at 13:00, because they fall into different intervals. 

.. image:: fixed-window.png
   :align: center

To avoid this behavior, consider using `Sliding Window Counters`_ described below.

However Fixed Windows require minimal information to be stored unlike Sliding Windows discussed next – only the timestamp of the active interval to wrap around clock skew issues on different servers and know when to reset the counter, and the counter itself. As per the logic of a primitive, no timestamps of the individual background job executions are stored.

Sliding Window Counters
~~~~~~~~~~~~~~~~~~~~~~~~

Sliding Window Counters also limit the number of background job executions over a certain time window. But unlike Fixed Windows, where the whole timeline is divided into large fixed intervals, intervals within a Sliding Window Counter (called "buckets") are more fine grained. A Sliding Window stores multiple buckets, and each bucket has its timestamp and execution counter. 

In the following example we are creating a Sliding Window Counter with a one-hour interval and 3 buckets in each interval, and rate limit of 4 executions. 

.. code-block:: csharp

   manager.AddOrUpdateSlidingWindow("dropbox", new SlidingWindowOptions(
       limit: 4,
       interval: TimeSpan.FromHours(1),
       buckets: 3));

After creating a window counter, we need to decorate the necessary background job methods with the ``SlidingWindowAttribute`` filter with the same resource identifier as in the above code snippet to tell the state changing pipeline to inject the throttling logic.

.. code-block:: csharp

   [SlidingWindow("dropbox")]
   public void ProcessFiles() { /* ... */ }

Each bucket participates in multiple intervals as shown in the image below, and the *no more than X executions* requirement is enforced for each of those intervals. So if we had 4 executions at 12:59, all background jobs at 13:00 will be throttled and delayed unlike in a Fixed Window Counter.

.. image:: sliding-window.png
   :align: center

But as we can see in the picture above, background jobs 6-9 will be delayed to 13:40 and executed successfully at that time, although the configured one-hour interval has not passed yet. We can increase the number of buckets to a higher value, but the minimum allowable interval of a single bucket is 1 second. 

.. note::

   So there is always a chance that limits will be violated, but that is a practical limitation – otherwise we will need to store timestamp for each individual background job that will result in an enormous payload size.

When it is time to remove the throttling on all the affected methods, just remove their references to the ``SlidingWindowAttribute`` filter and call the following method. Unlike `Concurrency Limiters`_ it is safe to remove the attributes without changing the mode first, because no work is actually done during the background job completion.

.. code-block:: csharp

   manager.RemoveSlidingWindowIfExists("dropbox");

Dynamic Window Counters
~~~~~~~~~~~~~~~~~~~~~~~

The Dynamic Window Counter allows us to create Sliding Window Counters dynamically depending on the background job arguments. It is also possible to set up an upper limit for all of its sliding windows, and even use some rebalancing strategies. With all of these features we can get some kind of fair processing, where one participant cannot capture all the available resources which is especially useful for multi-tenant applications.

``DynamicWindowAttribute`` filter is responsible for this kind of throttling, and along with setting a resource identifier we need to specify the window format with String.Format-like placeholders (as in `Mutexes`_) that will be converted into dynamic window identifiers at run-time based on job arguments. 

.. admonition:: Maximal length of resource identifiers is 100 characters
   :class: note

   Please keep this in mind especially when using dynamic resource identifiers.

.. code-block:: c#

   [DynamicWindow("newsletter", "tenant:{0}")]
   public void SendNewsletter(long tenantId, string template) { /* ... */ }

Dynamic Fixed Windows
+++++++++++++++++++++

The following code snippet demonstrates the simplest form of a Dynamic Window Counter. Since there is a single bucket, it will create a Fixed Window of one-hour length with a maximum of 4 executions per each tenant. There will be up to 1000 fixed windows so as not to blow up the data structure's size.

.. code-block:: c#

   IThrottlingManager manager = new ThrottlingManager();

   manager.AddOrUpdateDynamicWindow("newsletter", new DynamicWindowOptions(
       limit: 4,
       interval: TimeSpan.FromHours(1),
       buckets: 1));

Dynamic Sliding Windows
+++++++++++++++++++++++

If we increase the number of buckets, we will be able to use Sliding Windows instead with the given number of buckets. Limitations are the same as in Sliding Windows, so the minimum bucket length is 1 second. As with Fixed Windows, there can only be up to 1000 Sliding Windows in order to keep the size under control.

.. code-block:: c#

   manager.AddOrUpdateDynamicWindow("newsletter", new DynamicWindowOptions(
       limit: 4,
       interval: TimeSpan.FromHours(1),
       buckets: 60));

Limiting the Capacity
+++++++++++++++++++++

Capacity allows us to control how many fixed or sliding sub-windows will be created dynamically. After running the following sample, there will be a maximum 5 sub-windows limited to 4 executions. This is useful in scenarios when we do not want a particular background job to take all the available resources.

.. code-block:: c#

   manager.AddOrUpdateDynamicWindow("newsletter", new DynamicWindowOptions(
       capacity: 20,
       limit: 4,
       interval: TimeSpan.FromHours(1),
       buckets: 60));

Rebalancing Limits
++++++++++++++++++

When the capacity is set, we can also define dynamic limits for individual sub-windows in the following way. When rebalancing is enabled, individual limits depend on a number of active sub-windows and the capacity. 

.. code-block:: c#

   manager.AddOrUpdateDynamicWindow("newsletter", new DynamicWindowOptions(
       capacity: 20,
       minLimit: 2,
       maxLimit: 20,
       interval: TimeSpan.FromHours(1),
       buckets: 60));

So in the example above, if there are background jobs only for a single tenant, they will be performed at full speed, 20 per hour. But if other participants are trying to enter, existing ones will be limited in the following way. 

* 1 participant: 20 per hour
* 2 participants: 10 per hour for each
* 3 participants: 7 per hour for 2 of them, and 6 per hour for the last
* 4 participants: 5 per hour for each
* ...
* 10 participants: 2 per hour for each

Removing the Throttling
+++++++++++++++++++++++

As with other rate limiters, you can just remove the ``DynamicWindow`` attributes from your methods and call the following methods. There is no need to change the mode to ``Release`` as with `Concurrency Limiters`_, since no logic is running on background job completion.

.. code-block:: c#

   manager.RemoveDynamicWindowIfExists("newsletter");
