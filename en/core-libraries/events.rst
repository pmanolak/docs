Events System
#############

Creating maintainable applications is both a science and an art. It is
well-known that a key for having good quality code is making your objects
loosely coupled and strongly cohesive at the same time. Cohesion means that
all methods and properties for a class are strongly related to the class
itself and it is not trying to do the job other objects should be doing,
while loosely coupling is the measure of how little a class is "wired"
to external objects, and how much that class is depending on them.

There are certain cases where you need to cleanly communicate with other parts
of an application, without having to hard code dependencies, thus losing
cohesion and increasing class coupling. Using the Observer pattern, which allows
objects to notify other objects and anonymous listeners about changes is
a useful pattern to achieve this goal.

Listeners in the observer pattern can subscribe to events and choose to act upon
them if they are relevant. If you have used JavaScript, there is a good chance
that you are already familiar with event driven programming.

CakePHP emulates several aspects of how events are triggered and managed in
popular JavaScript libraries such as jQuery. In the CakePHP implementation, an
event object is dispatched to all listeners. The event object holds information
about the event, and provides the ability to stop event propagation at any
point. Listeners can register themselves or can delegate this task to other
objects and have the chance to alter the state and the event itself for the rest
of the callbacks.

The event subsystem is at the heart of Model, Behavior, Controller, View and
Helper callbacks. If you've ever used any of them, you are already somewhat
familiar with events in CakePHP.

Example Event Usage
===================

Let's suppose you are building a Cart plugin, and you'd like to focus on just
handling order logic. You don't really want to include shipping logic, emailing
the user or decrementing the item from the stock, but these are important tasks
to the people using your plugin. If you were not using events, you may try to
implement this by attaching behaviors to models, or adding components to your
controllers. Doing so represents a challenge most of the time, since you
would have to come up with the code for externally loading those behaviors or
attaching hooks to your plugin controllers.

Instead, you can use events to allow you to cleanly separate the concerns of
your code and allow additional concerns to hook into your plugin using events.
For example, in your Cart plugin you have an Orders model that deals with
creating orders. You'd like to notify the rest of the application that an order
has been created. To keep your Orders model clean you could use events::

    // Cart/Model/Table/OrdersTable.php
    namespace Cart\Model\Table;

    use Cake\Event\Event;
    use Cake\ORM\Table;

    class OrdersTable extends Table
    {
        public function place($order)
        {
            if ($this->save($order)) {
                $this->Cart->remove($order);
                $event = new Event('Order.afterPlace', $this, [
                    'order' => $order
                ]);
                $this->getEventManager()->dispatch($event);

                return true;
            }

            return false;
        }
    }

The above code allows you to notify the other parts of the application
that an order has been created. You can then do tasks like send email
notifications, update stock, log relevant statistics and other tasks in separate
objects that focus on those concerns.

Accessing Event Managers
========================

In CakePHP events are triggered against event managers. Event managers are
available in every Table, View and Controller using ``getEventManager()``::

    $events = $this->getEventManager();

Each model has a separate event manager, while the View and Controller
share one. This allows model events to be self contained, and allow components
or controllers to act upon events created in the view if necessary.

Global Event Manager
--------------------

In addition to instance level event managers, CakePHP provides a global event
manager that allows you to listen to any event fired in an application. This is
useful when attaching listeners to a specific instance might be cumbersome or
difficult. The global manager is a singleton instance of
:php:class:`Cake\\Event\\EventManager`. Listeners attached to the global
dispatcher will be fired before instance listeners at the same priority. You can
access the global manager using a static method::

    // In any configuration file or piece of code that executes before the event
    use Cake\Event\EventManager;

    EventManager::instance()->on(
        'Order.afterPlace',
        $aCallback
    );

One important thing you should consider is that there are events that will be
triggered having the same name but different subjects, so checking it in the
event object is usually required in any function that gets attached globally in
order to prevent some bugs. Remember that with the flexibility of using the
global manager, some additional complexity is incurred.

:php:meth:`Cake\\Event\\EventManager::dispatch()` method accepts the event
object as an argument and notifies all listener and callbacks passing this
object along. The listeners will handle all the extra logic around the
``afterPlace`` event, you can log the time, send emails, update user statistics
possibly in separate objects and even delegating it to offline tasks if you have
the need.

.. _tracking-events:

Tracking Events
---------------

To keep a list of events that are fired on a particular ``EventManager``, you
can enable event tracking. To do so, simply attach an
:php:class:`Cake\\Event\\EventList` to the manager::

    EventManager::instance()->setEventList(new EventList());

After firing an event on the manager, you can retrieve it from the event list::

    $eventsFired = EventManager::instance()->getEventList();
    $firstEvent = $eventsFired[0];

Tracking can be disabled by removing the event list or calling
:php:meth:`Cake\\Event\\EventList::trackEvents(false)`.

Core Events
===========

There are a number of core events within the framework which your application
can listen to. Each layer of CakePHP emits events that you can use in your
application.

* :ref:`ORM/Model events <table-callbacks>`
* :ref:`Controller events <controller-life-cycle>`
* :ref:`View events <view-events>`

``Server.terminate``
--------------------

The ``Server.terminate`` event is triggered after the response has been sent to the
client. This event is useful for performing tasks that should be done after the
response has been sent, such as logging or sending emails.

You can listen to this event using an event manager instance::

    use Cake\Event\EventManager;

    EventManager::instance()->on('Server.terminate', function ($event) {
        // Perform tasks that should be done after the response has been
        // sent to the client.
    });

Or using the ``events`` hook in your Application/Plugin class::

    use Cake\Event\EventManagerInterface;

    public function events(EventManagerInterface $eventManager): EventManagerInterface
    {
        $eventManager->on('Server.terminate', function ($event) {
            // Perform tasks that should be done after the response has been
            // sent to the client.
        });

        return $eventManager;
    }

.. tip::
    This is called even if an exception is thrown during the request, e.g. on 404 pages.

.. note::
    The ``Server.terminate`` event only works for PHP-FPM implementations which
    support the ``fastcgi_finish_request`` function.

.. _registering-event-listeners:

Registering Listeners
=====================

Listeners are the preferred way to register callbacks for an event. This is done
by implementing the :php:class:`Cake\\Event\\EventListenerInterface` interface
in any class you wish to register some callbacks. Classes implementing it need
to provide the ``implementedEvents()`` method. This method must return an
associative array with all event names that the class will handle.

To continue our previous example, let's imagine we have a UserStatistic class
responsible for calculating a user's purchasing history, and compiling into
global site statistics. This is a great place to use a listener class. Doing so
allows you to concentrate the statistics logic in one place and react to events
as necessary. Our ``UserStatistics`` listener might start out like::

    namespace App\Event;

    use Cake\Event\EventListenerInterface;

    class UserStatistic implements EventListenerInterface
    {
        public function implementedEvents(): array
        {
            return [
                // Custom event names let you design your application events
                // as required.
                'Order.afterPlace' => 'updateBuyStatistic',
            ];
        }

        public function updateBuyStatistic($event)
        {
            // Code to update statistics
        }
    }

    // From your controller, attach the UserStatistic object to the Order's event manager
    $statistics = new UserStatistic();
    $this->Orders->getEventManager()->on($statistics);

As you can see in the above code, the ``on()`` function will accept instances
of the ``EventListener`` interface. Internally, the event manager will use
``implementedEvents()`` to attach the correct callbacks.

.. versionadded:: 5.1.0
    The ``events`` hook was added to the ``BaseApplication`` as well as the ``BasePlugin`` class

As of CakePHP 5.1 it is recommended to register event listeners by adding them via the ``events`` hook in your application or plugin class::

    namespace App;

    use App\Event\UserStatistic;
    use Cake\Event\EventManagerInterface;
    use Cake\Http\BaseApplication;

    class Application extends BaseApplication
    {
        // The rest of your Application class

        public function events(EventManagerInterface $eventManager): EventManagerInterface
        {
            $statistics = new UserStatistic();
            $eventManager->on($statistics);

            return $eventManager;
        }
    }

Registering Anonymous Listeners
-------------------------------

While event listener objects are generally a better way to implement listeners,
you can also bind any ``callable`` as an event listener. For example if we
wanted to put any orders into the log files, we could use a simple anonymous
function to do so::

    use Cake\Log\Log;

    // From within a controller, or during application bootstrap.
    $this->Orders->getEventManager()->on('Order.afterPlace', function ($event) {
        Log::write(
            'info',
            'A new order was placed with id: ' . $event->getSubject()->id
        );
    });

In addition to anonymous functions you can use any other callable type that PHP
supports::

    $events = [
        'email-sending' => 'EmailSender::sendBuyEmail',
        'inventory' => [$this->InventoryManager, 'decrement'],
    ];
    foreach ($events as $callable) {
        $eventManager->on('Order.afterPlace', $callable);
    }

When working with plugins that don't trigger specific events, you can leverage
event listeners on the default events. Lets take an example  'UserFeedback'
plugin which handles feedback forms from users. From your application you would
like to know when a Feedback record has been saved and ultimately act on it. You
can listen to the global ``Model.afterSave`` event.  However, you can take
a more direct approach and only listen to the event you really need::

    // You can create the following before the
    // save operation, ie. config/bootstrap.php
    use Cake\Datasource\FactoryLocator;
    // If sending emails
    use Cake\Mailer\Email;

    FactoryLocator::get('Table')->get('ThirdPartyPlugin.Feedbacks')
        ->getEventManager()
        ->on('Model.afterSave', function($event, $entity)
        {
            // For example we can send an email to the admin
            $email = new Email('default');
            $email->setFrom(['info@yoursite.com' => 'Your Site'])
                ->setTo('admin@yoursite.com')
                ->setSubject('New Feedback - Your Site')
                ->send('Body of message');
        });

You can use this same approach to bind listener objects.

Interacting with Existing Listeners
-----------------------------------

Assuming several event listeners have been registered the presence or absence
of a particular event pattern can be used as the basis of some action.::

    // Attach listeners to EventManager.
    $this->getEventManager()->on('User.Registration', [$this, 'userRegistration']);
    $this->getEventManager()->on('User.Verification', [$this, 'userVerification']);
    $this->getEventManager()->on('User.Authorization', [$this, 'userAuthorization']);

    // Somewhere else in your application.
    $events = $this->getEventManager()->matchingListeners('Verification');
    if (!empty($events)) {
        // Perform logic related to presence of 'Verification' event listener.
        // For example removing the listener if present.
        $this->getEventManager()->off('User.Verification');
    } else {
        // Perform logic related to absence of 'Verification' event listener
    }

.. note::

    The pattern passed to the ``matchingListeners`` method is case sensitive.

.. _event-priorities:

Establishing Priorities
-----------------------

In some cases you might want to control the order that listeners are invoked.
For instance, if we go back to our user statistics example. It would be ideal if
this listener was called at the end of the stack. By calling it at the end of
the listener stack, we can ensure that the event was not cancelled, and that no
other listeners raised exceptions. We can also get the final state of the
objects in the case that other listeners have modified the subject or event
object.

Priorities are defined as an integer when adding a listener. The higher the
number, the later the method will be fired. The default priority for all
listeners is ``10``. If you need your method to be run earlier, using any value
below this default will work. On the other hand if you desire to run the
callback after the others, using a number above ``10`` will do.

If two callbacks happen to have the same priority value, they will be executed
with a the order they were attached. You set priorities using the ``on()``
method for callbacks, and declaring it in the ``implementedEvents()`` function
for event listeners::

    // Setting priority for a callback
    $callback = [$this, 'doSomething'];
    $this->getEventManager()->on(
        'Order.afterPlace',
        ['priority' => 2],
        $callback
    );

    // Setting priority for a listener
    class UserStatistic implements EventListenerInterface
    {
        public function implementedEvents()
        {
            return [
                'Order.afterPlace' => [
                    'callable' => 'updateBuyStatistic',
                    'priority' => 100
                ],
            ];
        }
    }

As you see, the main difference for ``EventListener`` objects is that you need
to use an array for specifying the callable method and the priority preference.
The ``callable`` key is a special array entry that the manager will read to know
what function in the class it should be calling.

Getting Event Data as Function Parameters
-----------------------------------------

When events have data provided in their constructor, the provided data is
converted into arguments for the listeners. An example from the View layer is
the afterRender callback::

    $this->getEventManager()
        ->dispatch(new Event('View.afterRender', $this, ['view' => $viewFileName]));

The listeners of the ``View.afterRender`` callback should have the following
signature::

    function (EventInterface $event, $viewFileName)

Each value provided to the Event constructor will be converted into function
parameters in the order they appear in the data array. If you use an associative
array, the result of ``array_values`` will determine the function argument
order.

.. note::

    Unlike in 2.x, converting event data to listener arguments is the default
    behavior and cannot be disabled.

Dispatching Events
==================

Once you have obtained an instance of an event manager you can dispatch events
using :php:meth:`~Cake\\Event\\EventManager::dispatch()`. This method takes an
instance of the :php:class:`Cake\\Event\\Event` class. Let's look at dispatching
an event::

    // An event listener has to be instantiated before dispatching an event.
    // Create a new event and dispatch it.
    $event = new Event('Order.afterPlace', $this, [
        'order' => $order
    ]);
    $this->getEventManager()->dispatch($event);

:php:class:`Cake\\Event\\Event` accepts 3 arguments in its constructor. The
first one is the event name, you should try to keep this name as unique as
possible, while making it readable. We suggest a convention as follows:
``Layer.eventName`` for general events happening at a layer level (for example,
``Controller.startup``, ``View.beforeRender``) and ``Layer.Class.eventName`` for
events happening in specific classes on a layer, for example
``Model.User.afterRegister`` or ``Controller.Courses.invalidAccess``.

The second argument is the ``subject``, meaning the object associated to the
event, usually when it is the same class triggering events about itself, using
``$this`` will be the most common case. Although a Component could trigger
controller events too. The subject class is important because listeners will get
immediate access to the object properties and have the chance to inspect or
change them on the fly.

Finally, the third argument is any additional event data. This can be any data
you consider useful to pass around so listeners can act upon it. While this can
be an argument of any type, we recommend passing an associative array.

The :php:meth:`~Cake\\Event\\EventManager::dispatch()` method accepts an event
object as an argument and notifies all subscribed listeners.

.. _stopping-events:

Stopping Events
---------------

Much like DOM events, you may want to stop an event to prevent additional
listeners from being notified. You can see this in action during model callbacks
(for example, beforeSave) in which it is possible to stop the saving operation if
the code detects it cannot proceed any further.

In order to stop events you can either return ``false`` in your callbacks or
call the ``stopPropagation()`` method on the event object::

    public function doSomething($event)
    {
        // ...
        return false; // Stops the event
    }

    public function updateBuyStatistic($event)
    {
        // ...
        $event->stopPropagation();
    }

Stopping an event will prevent any additional callbacks from being called.
Additionally the code triggering the event may behave differently based on the
event being stopped or not. Generally it does not make sense to stop 'after'
events, but stopping 'before' events is often used to prevent the entire
operation from occurring.

To check if an event was stopped, you call the ``isStopped()`` method in the
event object::

    public function place($order)
    {
        $event = new Event('Order.beforePlace', $this, ['order' => $order]);
        $this->getEventManager()->dispatch($event);
        if ($event->isStopped()) {
            return false;
        }
        if ($this->Orders->save($order)) {
            // ...
        }
        // ...
    }

In the previous example the order would not get saved if the event is stopped
during the ``beforePlace`` process.

Getting Event Results
---------------------

Every time a callback returns a non-null non-false value, it gets stored in the
``$result`` property of the event object. This is useful when you want to allow
callbacks to modify the event execution. Let's take again our ``beforePlace``
example and let callbacks modify the ``$order`` data.

Event results can be altered either using the event object result property
directly or returning the value in the callback itself::

    // A listener callback
    public function doSomething($event)
    {
        // ...
        $alteredData = $event->getData('order') + $moreData;

        return $alteredData;
    }

    // Another listener callback
    public function doSomethingElse($event)
    {
        // ...
        $event->setResult(['order' => $alteredData] + $this->result());
    }

    // Using the event result
    public function place($order)
    {
        $event = new Event('Order.beforePlace', $this, ['order' => $order]);
        $this->getEventManager()->dispatch($event);
        if (!empty($event->getResult()['order'])) {
            $order = $event->getResult()['order'];
        }
        if ($this->Orders->save($order)) {
            // ...
        }
        // ...
    }

It is possible to alter any event object property and have the new data passed
to the next callback. In most of the cases, providing objects as event data or
result and directly altering the object is the best solution as the reference is
kept the same and modifications are shared across all callback calls.

Removing Callbacks and Listeners
--------------------------------

If for any reason you want to remove any callback from the event manager just
call the :php:meth:`Cake\\Event\\EventManager::off()` method using as
arguments the first two parameters you used for attaching it::

    // Attaching a function
    $this->getEventManager()->on('My.event', [$this, 'doSomething']);

    // Detaching the function
    $this->getEventManager()->off('My.event', [$this, 'doSomething']);

    // Attaching an anonymous function.
    $myFunction = function ($event) { ... };
    $this->getEventManager()->on('My.event', $myFunction);

    // Detaching the anonymous function
    $this->getEventManager()->off('My.event', $myFunction);

    // Adding a EventListener
    $listener = new MyEventLister();
    $this->getEventManager()->on($listener);

    // Detaching a single event key from a listener
    $this->getEventManager()->off('My.event', $listener);

    // Detaching all callbacks implemented by a listener
    $this->getEventManager()->off($listener);

Events are a great way of separating concerns in your application and make
classes both cohesive and decoupled from each other. Events can be utilized to
de-couple application code and make extensible plugins.

Keep in mind that with great power comes great responsibility. Using too many
events can make debugging harder and require additional integration testing.

Additional Reading
==================

* :doc:`/orm/behaviors`
* :doc:`/console-commands/commands`
* :doc:`/controllers/components`
* :doc:`/views/helpers`
* :ref:`testing-events`

.. meta::
    :title lang=en: Events system
    :keywords lang=en: events, dispatch, decoupling, cakephp, callbacks, triggers, hooks, php
