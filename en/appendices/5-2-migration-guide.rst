5.2 Migration Guide
###################

The 5.2.0 release is a backwards compatible with 5.0. It adds new functionality
and introduces new deprecations. Any functionality deprecated in 5.x will be
removed in 6.0.0.

Behavior Changes
================

- ``ValidationSet::add()`` will now raise errors when a rule is added with
  a name that is already defined. This change aims to prevent rules from being
  overwritten by accident.
- ``Http\Session`` will now raise an exception when an invalid session preset is
  used.
- ``FormProtectionComponent`` now raises ``Cake\Controller\Exception\FormProtectionException``. This
  class is a subclass of ``BadRequestException``, and offers the benefit of
  being filterable from logging.
- ``NumericPaginator::paginate()`` now takes the ``finder`` option into account even when a ``SelectQuery`` instance is passed to it.

Deprecations
============

Console
-------

- ``Arguments::getMultipleOption()`` is deprecated. Use ``getArrayOption()``
  instead.


New Features
============

Console
-------

- The ``cake counter_cache`` command was added. This command can be used to
  regenerate counters for models that use ``CounterCacheBehavior``.
- ``ConsoleIntegrationTestTrait::debugOutput()`` makes it easier to debug
  integration tests for console commands.
- ``ConsoleInputArgument`` now supports a ``separator`` option. This option
  allows positional arguments to be delimited with a character sequence like
  ``,``. CakePHP will split the positional argument into an array when arguments
  are parsed.
- ``Arguments::getArrayArgumentAt()``, and ``Arguments::getArrayArgument()``
  were added. These methods allow you to read ``separator`` delimitered
  positional arguments as arrays.
- ``ConsoleInputOption`` now supports a ``separator`` option. This option
  allows option values to be delimited with a character sequence like
  ``,``. CakePHP will split the option value into an array when arguments
  are parsed.
- ``Arguments::getArrayArgumentAt()``, ``Arguments::getArrayArgument()``, and
  ``Arguments::getArrayOption()``
  were added. These methods allow you to read ``separator`` delimitered
  positional arguments as arrays.

ORM
---

- ``CounterCacheBehavior::updateCounterCache()`` has been addded. This method
  allows you to update the counter cache values for all records of the configured
  associations.

Error
-----

- Custom exceptions can have specific error handling logic defined in
  ``ErrorController``.
