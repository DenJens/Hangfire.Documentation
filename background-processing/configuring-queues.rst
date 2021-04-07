
Configuring Job Queues
======================

Hangfire can process multiple queues. If you want to prioritize your jobs, or split the processing across your servers (some processes for the archive queue, others for the images queue, etc), you can tell Hangfire about your decisions.

To place a job into a different queue, use the ``QueueAttribute`` class on your method:

.. code-block:: c#

   [Queue("alpha")]
   public void SomeMethod() { }

   BackgroundJob.Enqueue(() => SomeMethod());
  
.. admonition:: Queue name argument formatting 
   :class: warning

   The Queue name argument (since 1.7.6) can only consist of the following characters : lowercase letters, digits, underscores, and dashes.
  
To begin processing multiple queues, you need to update your ``BackgroundJobServer`` configuration.

.. code-block:: c#

   var options = new BackgroundJobServerOptions 
   {
       Queues = new[] { "alpha", "beta", "default" }
   };
   
   app.UseHangfireServer(options);
   // or
   using (new BackgroundJobServer(options)) { /* ... */ }

.. admonition:: Processing order
   :class: note

   Queues are run in the order that depends on the concrete storage implementation. For example, when we are using *Hangfire.SqlServer* the order is defined by alphanumeric order and the array index is ignored. When using *Hangfire.Pro.Redis* package, the array index is important and queues with a lower index will be processed first.

   The example above shows a generic approach, where workers will fetch jobs from the ``alpha`` queue first, ``beta`` second, and then from the ``default`` queue, regardless of the implementation.

ASP.NET Core
------------

For ASP.NET Core, define the queues array with ``services.AddHangfireServer`` in ``Startup.cs``:

.. code-block:: c#

   public void ConfigureServices(IServiceCollection services)
   {
       // Add the processing server as IHostedService
       services.AddHangfireServer(options =>
       {
           options.Queues = new[] { "alpha", "beta", "default" };
       });
   }
