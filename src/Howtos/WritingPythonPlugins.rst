=================================
How to write an S2E Python plugin
=================================

In this tutorial, we will look at how to write the plugin described `here <WritingPlugins.rst>`_ in Python. For details
on how S2E supports Python plugins, read `this <src/PythonPluginArchitecture.rst>`_.

Starting with an empty plugin
=============================

The first thing to do is to name the plugin and create boilerplate code. Let us name the plugin ``InstructionTracer``.
This plugin must be located within a Python module, so create a directory structure similar to the following:

::

    s2e_python_plugins/
    ├── __init__.py
    ├── instruction_tracker.py

Put the following content inside ``instruction_tracker.py``:

.. code-block:: python

    # These modules are exported by S2E's Python interpreter
    from s2e.core.plugins import Plugin
    from s2e.core import signals

    class InstructionTracker(Python):
        description = 'Tutorial - Tracking instructions'

        # Plugin dependencies would normally go here. However this plugin does not have any dependencies
        # dependencies = []

        def initialize(self):
            pass

Reading configuration parameters
================================

We would like to let the user specify which instruction to monitor. For this, we create a configuration variable that
stores the address of that instruction. Every plugin can have an entry in the S2E configuration file. The entry for our
plugin would look like this:

.. code-block:: lua

    pluginsConfig.InstructionTracker = {
        -- The location of the Python class within the module
        pythonClass = "instruction_tracker.InstructionTracker",
        -- The address we want to track
        addressToTrack = 0x12345,
    }

We also need to add our plugin to the ``Plugins`` table. This can be done one of two ways:

* If using ``s2e-env``, add a call to ``add_plugin`` to ``s2e-config.lua``.

.. code-block:: lua

    add_plugin("InstructionTracker")

* If not using ``s2e-env``, you will have to create the ``Plugins`` table yourself:

.. code-block:: lua

    Plugins = {
        -- List other plugins here
        "InstructionTracker",
    }

Finally, we must tell S2E where to look for Python modules. We do this under the ``s2e.python.modulePaths`` table:

.. code-block:: lua

    s2e = {
        python = {
            modulePaths = {
                "/path/to/s2e_python_plugins",
            },
        },
    }

If we run the plugin as it is now, nothing will happen. S2E ignores any unknown configuration value. We need a
mechanism to explicitly retrieve the configuration value. In S2E, plugins can retrieve the configuration at any time.
In our case, we do it during the initialization phase.

.. code-block:: python

    # ...

    def __init__(self):
        super(InstructionTracker, self).__init__()

        # Although not strictly required, it is good Python programming practice to initialize instance
        # attributes in the class constructor
        self._address = None

    def initialize(self):
        self._address = self.config['addressToTrack']

Instrumenting instructions
==========================

To instrument an instruction, an S2E plugin registers to the ``on_translate_instruction_start`` core event. There are
many core events to which a plugin can register. These events are defined in ``CorePlugin.h`` and exposed to S2E's
Python interpreter in ``export_signals.cpp``. Both of these files can be found in the `libs2ecore
<https://github.com/S2E/libs2ecore>`_ repository.

Extend your code as follows.

.. code-block:: python

    # ...

    def initialize(self):
        self._address = self.config['addressToTrack']

        # This indicates that our plugin is interested in monitoring instruction translation.
        # For this, the plugin registers a callback with the on_translate_instruction_start signal
        signals.on_translate_instruction_start.connect(self._on_translate_instruction)

    def _on_translate_instruction(self, signal, state, tb, pc):
        if self._address == pc:
            # When we find an interesting address, ask S2E to invoke our callback when the address is actually
            # executed
            signal.connect(self._on_instruction_execution)

    # This callback is called only when the instruction at our address is executed.
    # The callback incurs zero overhead for all other instructions
    def _on_instruction_execution(self, state, pc):
        self.debug('Executing instruction at 0x%x' % pc)
        # The plugins can arbitrarily modify/observe the current execution state via the state parameter.
        # Plugins can also access self.s2e to use the S2E API

Counting instructions
=====================

We would like to count how many times that particular instruction is executed. There are two options:

1. Count how many times it was executed across all paths
2. Count how many times it was executed in each path

The first option is trivial to implement. Simply add an additional member to the class and increment it every time the
``_on_instruction_execution`` callback is invoked.

The second option requires to keep per-state plugin information. Unlike C++ plugin state, we can store Python plugin
state in any type, as long as it is stored within the plugin's ``_state`` attribute.

Here is how ``InstructionTracker`` could implement the plugin state.

.. code-block:: python

    class InstructionTrackerState(object):
        def __init__(self):
            self._count = 0

        def increment(self):
            self._count += 1

        @property
        def count(self):
            return self._count

Plugin code can refer to this state using the ``_state`` attribute:

.. code-block:: python

    def _on_instruction_execution(self, state, pc):
        self.debug('Executing instruction at 0x%x' % pc)

        # Increment the count
        self._state.increment()

Exporting events
================

Python plugins do not yet support defining custom events.
