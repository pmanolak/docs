Appendices
##########

Appendices contain information regarding the new features
introduced in each version and the migration path between versions.

Migration Guides
================

:doc:`appendices/migration-guides`

Backwards Compatibility Shimming
================================

If you need/want to shim 4.x behavior, or partially migrate in steps, check out
the `Shim plugin <https://github.com/dereuromark/cakephp-shim>`__ that can help mitigate some BC breaking changes.

Forwards Compatibility Shimming
===============================

Forwards compatibility shimming can prepare your 4.x app for the next major
release (5.x).

If you already want to shim 5.x behavior into 4.x, check out the `Shim plugin
<https://github.com/dereuromark/cakephp-shim>`__. This plugin aims to mitigate
some backwards compatibility breakage and help backport features from 5.x to
4.x.  The closer your 3.x app is to 4.x, the smaller will be the diff of
changes, and the smoother will be the final upgrade.

General Information
===================

.. toctree::
    :maxdepth: 1

    appendices/cakephp-development-process
    appendices/glossary

.. meta::
    :title lang=en: Appendices
    :keywords lang=en: migration guide,migration path,new features,glossary
