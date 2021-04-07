Dealing with exceptions
========================

Bad things happen. Any method can throw different types of exceptions. These exceptions can be caused either by programming errors that require you to re-deploy the application, or transient errors, that can be fixed without additional deployment.

Hangfire handles all exceptions that occurred within both internal (belonging to Hangfire itself), and external methods (jobs, filters and so on), so it will not bring down the whole application. All internal exceptions are logged (so, do not forget to :doc:`enable logging <../configuration/configuring-logging>`) and the worst case they can lead to is – background processing will be stopped after ``10`` retry attempts with an increasing delay modifier.

When Hangfire encounters an external exception that occurred during the job performance, it will automatically *try* to change its state to the ``Failed`` one, and thus you can always find this job in the Monitor UI (it will not be expired unless you delete it explicitly).

.. image:: failed-job.png

In the previous paragraph I said that Hangfire *will try* to change its state to failed, because state transition is one of places, where :doc:`job filters <../extensibility/using-job-filters>` can intercept and change the initial pipeline. And the ``AutomaticRetryAttribute`` class is one of them, that schedules the failed job to be automatically retried after increasing delay.

This filter is applied globally to all methods and grants 10 retry attempts by default. So, your methods will be retried in case of exception automatically, and you receive warning log messages on every failed attempt. If the number of retry attempts exceed their maximum, the job will be move to the ``Failed`` state (with an error log message), and you will be able to retry it manually.

If you do not want a job to be retried, place an explicit attribute with 0 maximum retry attempts as its value:

.. code-block:: c#

   [AutomaticRetry(Attempts = 0)]
   public void BackgroundMethod()
   {   
   }

Use the same way to limit the number of attempts to the different value. If you want to change the default global value, add a new global filter:

.. code-block:: c#

   GlobalJobFilters.Filters.Add(new AutomaticRetryAttribute { Attempts = 5 });


If you are using ASP.NET Core you can use the IServiceCollection extension method AddHangfire. Note that AddHangfire uses the GlobalJobFilter instance and therefore dependencies should be Transient or Singleton.

.. code-block:: c#

    services.AddHangfire((provider, configuration) =>
    {
        configuration.UseFilter(provider.GetRequiredService<AutomaticRetryAttribute>());
    }
