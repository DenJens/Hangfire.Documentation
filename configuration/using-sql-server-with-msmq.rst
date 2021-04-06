Using SQL Server with MSMQ
===========================

`Hangfire.SqlServer.MSMQ <https://www.nuget.org/packages/Hangfire.SqlServer.MSMQ/>`_ extension changes the way Hangfire handles job queues. Default :doc:`implementation <using-sql-server>` uses regular SQL Server tables to organize queues, and this extensions uses transactional MSMQ queues to process jobs. Please note that starting from 1.7.0 it is possible to use ``TimeSpan.Zero`` as a polling delay in Hangfire.SqlServer, so think twice before using MSMQ.

Installation
-------------

MSMQ support for SQL Server job storage implementation, like other Hangfire extensions, is a NuGet package. So, you can install it using NuGet Package Manager Console window:

.. code-block:: powershell

   PM> Install-Package Hangfire.SqlServer.Msmq

Configuration
--------------

To use MSMQ queues, you should do the following steps:

1. **Create them manually on each host**. Do not forget to grant appropriate permissions. Please note that queue storage is limited to 1048576 KB by default (approximately 2 millions enqueued jobs), you can increase it through the MSMQ properties window. 
2. Register all MSMQ queues in current ``SqlServerStorage`` instance.

If you are using **only default queue**, call the ``UseMsmqQueues`` method just after ``UseSqlServerStorage`` method call and pass the path pattern as an argument.

.. code-block:: c#

    GlobalConfiguration.Configuration
        .UseSqlServerStorage("<connection string or its name>")
        .UseMsmqQueues(@"FormatName:Direct=OS:localhost\hangfire-{0}");

To use multiple queues, you should pass them explicitly:

.. code-block:: c#

    GlobalConfiguration.Configuration
        .UseSqlServerStorage("<connection string or its name>")
        .UseMsmqQueues(@"FormatName:Direct=OS:localhost\hangfire-{0}", "critical", "default");

Limitations
------------

* Only transactional MSMQ queues are supported for reliability reasons inside ASP.NET.
* You cannot use both SQL Server Job Queue and MSMQ Job Queue implementations in the same server (see below). This limitation relates to Hangfire Server only. You can still enqueue jobs to whatever queues and watch them both in Hangfire Dashboard.

Transition to MSMQ queues
--------------------------

If you have a fresh installation, just use the ``UseMsmqQueues`` method. Otherwise, your system may contain unprocessed jobs in SQL Server. Since one Hangfire Server instance cannot process jobs from different queues, you should deploy :doc:`multiple instances <../background-processing/running-multiple-server-instances>` of Hangfire Server, one listens only MSMQ queues, another – only SQL Server queues. When the latter finishes its work (you can see this in Dashboard – your SQL Server queues will be removed), you can remove it safely.

If you are using the default queue only, do this:

.. code-block:: c#

    /* This server will process only SQL Server table queues, i.e. old jobs */
    var oldStorage = new SqlServerStorage("<connection string or its name>");
    var oldOptions = new BackgroundJobServerOptions
    {
        ServerName = "OldQueueServer" // Pass this to differentiate this server from the next one
    };

    app.UseHangfireServer(oldOptions, oldStorage);

    /* This server will process only MSMQ queues, i.e. new jobs */
    GlobalConfiguration.Configuration
        .UseSqlServerStorage("<connection string or its name>")
        .UseMsmqQueues(@"FormatName:Direct=OS:localhost\hangfire-{0}");

    app.UseHangfireServer();

If you use multiple queues, do this:

.. code-block:: c#

    /* This server will process only SQL Server table queues, i.e. old jobs */
    var oldStorage = new SqlServerStorage("<connection string>");
    var oldOptions = new BackgroundJobServerOptions
    {
        Queues = new [] { "critical", "default" }, // Include this line only if you have multiple queues
        ServerName = "OldQueueServer" // Pass this to differentiate this server from the next one
    };

    app.UseHangfireServer(oldOptions, oldStorage);

    /* This server will process only MSMQ queues, i.e. new jobs */
    GlobalConfiguration.Configuration
        .UseSqlServerStorage("<connection string or its name>")
        .UseMsmqQueues(@"FormatName:Direct=OS:localhost\hangfire-{0}", "critical", "default");

    app.UseHangfireServer();
