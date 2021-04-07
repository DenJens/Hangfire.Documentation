Passing dependencies
=======================

In almost every job you will want to use other classes of your application to perform different work and keep your code clean and simple. Let us call these classes as *dependencies*. How do we pass these dependencies to methods that will be called in the background?

When you are calling static methods in the background, you are restricted only to the static context of your application, and this requires you to use the following patterns of obtaining dependencies:

* Manual dependency instantiation through the ``new`` operator
* `Service location <http://en.wikipedia.org/wiki/Service_locator_pattern>`_
* `Abstract factories <http://en.wikipedia.org/wiki/Abstract_factory_pattern>`_ or `builders <http://en.wikipedia.org/wiki/Builder_pattern>`_
* `Singletons <http://en.wikipedia.org/wiki/Singleton_pattern>`_

However, all of these patterns greatly complicate the unit testability aspect of your application. To fight with this issue, Hangfire allows you to call instance methods in the background. Consider you have the following class that uses some kind of ``DbContext`` to access the database, and ``EmailService`` to send emails.

.. code-block:: c#

    public class EmailSender
    {
        public void Send(int userId, string message) 
        {
            var dbContext = new DbContext();
            var emailService = new EmailService();

            // Some processing logic
        }
    }

To call the ``Send`` method in background, use the following override of the ``Enqueue`` method (other methods of ``BackgroundJob`` class provide such overloads as well):

.. code-block:: c#

   BackgroundJob.Enqueue<EmailSender>(x => x.Send(13, "Hello!"));

When a worker determines that it needs to call an instance method, it creates the instance of a given class first using the current ``JobActivator`` class instance. By default, it uses the ``Activator.CreateInstance`` method that can create an instance of your class using **its default constructor**, so let us add that in:

.. code-block:: c#

   public class EmailSender
   {
       private IDbContext _dbContext;
       private IEmailService _emailService;

       public EmailSender()
       {
           _dbContext = new DbContext();
           _emailService = new EmailService();
       } 

       // ...
   }

If you want the class to be ready for unit testing, consider adding a constructor overload, because the **default activator cannot create an instance of a class that has no default constructor**:

.. code-block:: c#

    public class EmailSender
    {
        // ...

        public EmailSender()
            : this(new DbContext(), new EmailService())
        {
        }

        internal EmailSender(IDbContext dbContext, IEmailService emailService)
        {
            _dbContext = dbContext;
            _emailService = emailService;
        }
    }

If you are using IoC containers, such as Autofac, Ninject, SimpleInjector and so on, you can remove the default constructor. To learn how to do this, proceed to the next section.
