Command Lifecycle Events
########################

.. php:namespace:: Cake\Console

CakePHP commands trigger events during execution that allow you to hook into
different stages of the command lifecycle. These events work similarly to
controller lifecycle callbacks.

Event List
==========

The following events are triggered during command execution:

* ``Command.beforeExecute`` - Before the ``execute()`` method
* ``Command.afterExecute`` - After the ``execute()`` method completes

Lifecycle Callback Methods
==========================

beforeExecute()
---------------

.. php:method:: beforeExecute(EventInterface $event)

Called before the ``execute()`` method runs. Useful for initialization and
validation::

    use Cake\Event\EventInterface;

    class MyCommand extends Command
    {
        public function beforeExecute(EventInterface $event): void
        {
            parent::beforeExecute($event);

            $this->log('Starting command execution');

            if (!$this->checkPrerequisites()) {
                $event->stopPropagation();
                $this->io()->error('Prerequisites not met');
            }
        }
    }

afterExecute()
--------------

.. php:method:: afterExecute(EventInterface $event)

Called after the ``execute()`` method completes. Useful for cleanup and
logging::

    public function afterExecute(EventInterface $event): void
    {
        parent::afterExecute($event);

        $this->cleanup();
        $this->log('Command execution completed');
    }

Stopping Command Execution
==========================

You can prevent the ``execute()`` method from running by stopping event
propagation in ``beforeExecute()``::

    public function beforeExecute(EventInterface $event): void
    {
        if (!$this->isValid()) {
            $event->stopPropagation();
            $event->setResult(static::CODE_ERROR);
        }
    }

When propagation is stopped, ``execute()`` won't run but ``afterExecute()``
will still be called for cleanup.

.. note::

    Remember to call ``parent::beforeExecute($event)`` and
    ``parent::afterExecute($event)`` when overriding these methods.