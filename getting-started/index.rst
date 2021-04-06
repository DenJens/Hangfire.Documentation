Getting Started
================

Requirements
-------------

Hangfire works with the majority of .NET platforms: .NET Framework 4.5 or later, .NET Core 1.0 or later, or any platform compatible with .NET Standard 1.3. You can integrate it with almost any application framework, including ASP.NET, ASP.NET Core, Console applications, Windows Services, WCF, as well as community-driven frameworks like Nancy or ServiceStack.

Storage
--------

Storage is the place where Hangfire keeps all the information related to background job processing. All the details like types, method names, arguments, etc. are serialized and placed into storage, no data is kept in process memory. The storage subsystem is abstracted in Hangfire well enough to be implemented within RDBMS and NoSQL solutions.

This is the main decision you must make, and the only configuration required before starting to use the framework. The following example shows how to configure Hangfire with a SQL Server database. Please note that the connection string may vary, depending on your environment.

.. code-block:: csharp

   GlobalConfiguration.Configuration
       .UseSqlServerStorage(@"Server=.\SQLEXPRESS; Database=Hangfire.Sample; Integrated Security=True");

Client
-------

The Client is responsible for creating background jobs and saving them into Storage. A background job is a unit of work that should be performed outside of the current execution context, e.g. in background thread, other process, or even on a different server – all this is possible with Hangfire, even with no additional configuration.

.. code-block:: csharp

   BackgroundJob.Enqueue(() => Console.WriteLine("Hello, world!"));

Please note this is not a delegate, it's an *expression tree*. Instead of calling the method immediately, Hangfire serializes the type (``System.Console``), method name (``WriteLine``, with all the parameter types to identify it later), and all the given arguments, and places it into Storage.

Server
-------

The Hangfire Server processes background jobs by querying the Storage. Roughly speaking, it is a set of background threads that listen to the Storage for new background jobs, and performs them by de-serializing the type, method and arguments.

You can place this background job server in any process you want, including `dangerous ones <http://haacked.com/archive/2011/10/16/the-dangers-of-implementing-recurring-background-tasks-in-asp-net.aspx/>`_ like ASP.NET – even if you terminate a process, your background jobs will be retried automatically after restart. So in a basic configuration for a web application, you do not need to use Windows Services for background processing anymore.

.. code-block:: csharp

   using (new BackgroundJobServer())
   {
       Console.ReadLine();
   }

Installation
-------------

Hangfire is distributed as a couple of NuGet packages, starting from the primary one, Hangfire.Core, that contains all the primary classes as well as abstractions. Other packages like Hangfire.SqlServer provide features or abstraction implementations. To start using Hangfire, install the primary package and choose one of the :doc:`available storages </storages/index>`.

After the release of Visual Studio 2017, a completely new way of installing NuGet packages appeared. So I gave up listing all the ways of installing a NuGet package, and fellback to the one available almost everywhere using the ``dotnet`` app.

.. code:: bash

   dotnet add package Hangfire.Core
   dotnet add package Hangfire.SqlServer

Configuration
--------------

Configuration is performed using the ``GlobalConfiguration`` class. Its ``Configuration`` property provides a lot of extension methods, both from Hangfire.Core, as well as other packages. If you install a new package, do not hesitate to check whether there are new extension methods. 

.. code-block:: c#

   GlobalConfiguration.Configuration
       .SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
       .UseSimpleAssemblyNameTypeSerializer()
       .UseRecommendedSerializerSettings()
       .UseSqlServerStorage("Database=Hangfire.Sample; Integrated Security=True;", new SqlServerStorageOptions
       {
           CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
           SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
           QueuePollInterval = TimeSpan.Zero,           
           UseRecommendedIsolationLevel = true,
           UsePageLocksOnDequeue = true,
           DisableGlobalLocks = true
       })
       .UseBatches()
       .UsePerformanceCounters();

Method calls can be chained, so there is no need to use the class name again and again. Global configuration is made for simplicity, almost every class of Hangfire allows you to specify overrides for storage, filters, etc. In ASP.NET Core environment's global configuration class is hidden inside the ``AddHangfire`` method.

Usage
------

Here are all the Hangfire components in action, as a fully working sample that prints the "Hello, world!" message from a background thread. You can comment out the lines related to server, and run the program several times – all the background jobs will be processed as soon as you uncomment out the lines again.

.. code-block:: c#

   using System;
   using Hangfire;
   using Hangfire.SqlServer;

   namespace ConsoleApplication2
   {
       class Program
       {
           static void Main()
           {
               GlobalConfiguration.Configuration
                   .SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
                   .UseColouredConsoleLogProvider()
                   .UseSimpleAssemblyNameTypeSerializer()
                   .UseRecommendedSerializerSettings()
                   .UseSqlServerStorage("Database=Hangfire.Sample; Integrated Security=True;", new SqlServerStorageOptions
                   {
                       CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
                       SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
                       QueuePollInterval = TimeSpan.Zero,           
                       UseRecommendedIsolationLevel = true,
                       UsePageLocksOnDequeue = true,
                       DisableGlobalLocks = true
                   });

               BackgroundJob.Enqueue(() => Console.WriteLine("Hello, world!"));

               using (var server = new BackgroundJobServer())
               {
                   Console.ReadLine();
               }
           }
       }
   }

.. toctree::
   :maxdepth: 1
   :hidden:

   aspnet-applications
   aspnet-core-applications
