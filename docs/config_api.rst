.. _config_api:

================================
Configuration File API Reference
================================

This document describes the low-level API for ``PyOxidizer`` configuration
files. For a higher-level overview of how configuration files work, see
:ref:`config_files`.

Global Symbols
==============

The following are all global symbols available by default in the
Starlark environment:

* `Starlark built-ins <https://github.com/bazelbuild/starlark/blob/master/spec.md#built-in-constants-and-functions>`_.
* :ref:`config_build_target_triple`
* :ref:`config_config_path`
* :ref:`config_context`
* :ref:`config_cwd`
* :ref:`config_default_python_distribution`
* :ref:`config_file_manifest`
* :ref:`config_glob`
* :ref:`config_python_bytecode_module`
* :ref:`config_python_distribution`
* :ref:`config_python_embedded_data`
* :ref:`config_python_executable`
* :ref:`config_python_extension_module`
* :ref:`config_python_interpreter_config`
* :ref:`config_python_resources_data`
* :ref:`config_python_source_module`
* :ref:`config_register_target`
* :ref:`config_resolve_target`
* :ref:`config_resolve_targets`
* :ref:`config_set_build_path`

Types
=====

The following custom data types are defined in the Starlark environment:

``FileManifest``
   Represents a mapping of filenames to file content.

``PythonBytecodeModule``
   Represents a ``.pyc`` file containing Python bytecode for a given module.

``PythonDistribution``
   Represents an implementation of Python.

   Used for embedding into binaries and running Python code.

``PythonEmbeddedData``
   Represents resources embedded in a binary to define and run a Python
   interpreter.

``PythonExecutable``
   Represents an executable file containing a Python interpreter.

``PythonExtensionModule``
   Represents a compiled Python extension module.

``PythonInterpreterConfig``
   Represents the configuration of a Python interpreter.

``PythonResourcesData``
   Represents a non-module *resource* data file.

``PythonSourceModule``
   Represents a ``.py`` file containing Python source code.

Constants
=========

PyOxidizer provides global constants as defined by the following sections.

.. _config_build_target_triple:

BUILD_TARGET_TRIPLE
-------------------

The string Rust target triple that we're currently building for. Will be
a value like ``x86_64-unknown-linux-gnu`` or ``x86_64-pc-windows-msvc``.
Run ``rustup target list`` to see a list of targets.

.. _config_config_path:

CONFIG_PATH
-----------

The string path to the configuration file currently being evaluated.

.. _config_context:

CONTEXT
-------

Holds build context. This is an internal variable and accessing it will
not provide any value.

.. _config_cwd:

CWD
---

The current working directory. Also the directory containing the active
configuration file.

Functions for Manipulating Global State
=======================================

.. _config_set_build_path:

set_build_path(path)
--------------------

Configure the directory where build artifacts will be written.

Build artifacts include Rust build state, files generated by PyOxidizer,
staging areas for built binaries, etc.

If a relative path is passed, it is interpreted as relative to the
directory containing the configuration file.

The default value is ``$CWD/build``.

.. important::

   This needs to be called before functionality that utilizes the build path,
   otherwise the default value will be used.

Functions for Managing Targets
==============================

.. _config_register_target:

register_target(name, fn, depends=[], default=False, default_build_script=False)
--------------------------------------------------------------------------------

Registers a named target that can be resolved by the configuration file.

A target consists of a string name, callable function, and an optional list
of targets it depends on.

The callable may return one of the types defined by this Starlark dialect
to facilitate additional behavior, such as how to build and run it.

``depends`` is an optional list of target strings this target depends on.
If specified, each dependency will be evaluated in order and its returned
value (possibly cached from prior evaluation) will be passed as a
positional argument to this target's callable.

``default`` indicates whether this should be the default target
to evaluate. The last registered target setting this to ``True``
will be the default. If no target sets this to ``True``, the first
registered target is the default.

``default_build_script`` indicates whether this should be the default
target to evaluate when run from the context of a Rust build script (e.g.
from ``pyoxidizer run-build-script``. It has the same semantics as
``default``.

.. note::

   It would be easier for target functions to call ``resolve_target()``
   within their implementation. However, Starlark doesn't allow recursive
   function calls. So invocation of target callables must be handled
   specially to avoid this recursion.

.. _config_resolve_target:

resolve_target(target)
----------------------

Triggers resolution of a requested build target.

This function resolves a target registered with ``register_target()`` by
calling the target's registered function or returning the previously
resolved value from calling it.

This function should be used in cases where 1 target depends on the
resolved value of another target. For example, a target to create a
``FileManifest`` may wish to add a ``PythonExecutable`` that was resolved
from another target.

.. _config_resolve_targets:

resolve_targets()
-----------------

Triggers resolution of requested build targets.

This is usually the last meaningful line in a config file. It triggers the
building of targets which have been requested to resolve by whatever is invoking
the config file.

.. _config_python_distributions:

Python Distributions
====================

Python distributions are entities that define an implementation of Python
that can be used to create a binary embedding Python and that can be used
to execute Python code.

Python distributions are defined by the ``PythonDistribution`` type. This
type can be constructed from parameters or via
:ref:`config_default_python_distribution`.

.. _config_python_distribution:

``PythonDistribution(sha256, local_path=None, url=None, flavor="standalone")``
------------------------------------------------------------------------------

Defines a Python distribution that can be embedded into a binary.

A Python distribution is a zstandard-compressed tar archive containing a
specially produced build of Python. These distributions are typically
produced by the
`python-build-standalone <https://github.com/indygreg/python-build-standalone>`_
project. Pre-built distributions are available at
https://github.com/indygreg/python-build-standalone/releases.

A distribution is defined by a location, and a hash.

One of ``local_path`` or ``url`` MUST be defined.

``sha256`` (string)
   The SHA-256 of the distribution archive file.

``local_path`` (string)
   Local filesystem path to the distribution archive.

``url`` (string)
   URL from which a distribution archive can be obtained using an HTTP GET
   request.

``flavor`` (string)
   The distribution flavor. Can either by ``standalone`` (the default) or
   ``windows_embeddable``.

Examples:

.. code-block:: python

   linux = PythonDistribution(
       sha256="11a53f5755773f91111a04f6070a6bc00518a0e8e64d90f58584abf02ca79081",
       local_path="/var/python-distributions/cpython-linux64.tar.zst"
   )

   macos = PythonDistribution(
        sha256="b46a861c05cb74b5b668d2ce44dcb65a449b9fef98ba5d9ec6ff6937829d5eec",
        url="https://github.com/indygreg/python-build-standalone/releases/download/20190505/cpython-3.7.3-macos-20190506T0054.tar.zst"
   )


.. _config_default_python_distribution:

``default_python_distribution(flavor="standalone", build_target=None)``
-----------------------------------------------------------------------

Resolves the default ``PythonDistribution`` for the given distribution
flavor and build target, which default to a ``standalone`` distribution and
the active build target as defined by ``BUILD_TARGET``, respectively.

``flavor`` is a string denoting the distribution *flavor*. Values can be one
of the following:

``standalone``
   A distribution produced by the ``python-build-standalone`` project. These
   distributions allow producing a self-contained, statically linked binary
   embedding Python.

``windows_embeddable``
   A Windows-only distribution format defined by a zip file. These distributions
   are produced by the official Python project. **Support for this distribution
   flavor is experimental, doesn't fully work, and may be removed in a future
   release because it may not be viable.**

The ``pyoxidizer`` binary has a set of known distributions built-in
which are automatically available and used by this function. Typically you don't
need to build your own distribution or change the distribution manually.

``PythonDistribution`` Methods
------------------------------

``PythonDistribution.source_modules()``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Returns a ``list`` of ``PythonSourceModule`` representing Python
source modules present in this distribution.

``PythonDistribution.resources_data(include_test=False)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Returns a ``list`` of ``PythonResourceData`` representing resource files
present in this distribution.

The ``include_test`` boolean argument controls whether resources associated
with test packages are included.

.. _config_python_distribution_extension_modules:

``PythonDistribution.extension_modules(filter='all', preferred_variants=None)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Returns a ``list`` of ``PythonExtensionModule`` representing extension
modules in this distribution.

The ``filter`` argument denotes how to filter the extension modules. The
following values are recognized:

``all``
   Every named extension module will be included.

``minimal``
   Return only extension modules that are required to initialize a
   Python interpreter. This is a very small set and various functionality
   from the Python standard library will not work with this value.

``no-libraries``
   Return only extension modules that don't require any additional libraries.

   Most common Python extension modules are included. Extension modules
   like ``_ssl`` (links against OpenSSL) and ``zlib`` are not included.

``no-gpl``
   Return only extension modules that do not link against GPL licensed
   libraries.

   Not all Python distributions may annotate license info for all extensions or
   the libraries they link against. If license info is missing, the extension is
   not included because it *could* be GPL licensed. Similarly, the mechanism for
   determining whether a license is GPL is based on an explicit list of non-GPL
   licenses. This ensures new GPL licenses don't slip through.

The ``preferred_variants`` argument denotes a string to string mapping of
extension module name to its preferred variant name. If multiple variants of
an extension module meet the filter requirements, the preferred variant from
this mapping will be used. Otherwise the first variant will be used.

.. important::

   Libraries that extension modules link against have various software
   licenses, including GPL version 3. Adding these extension modules will
   also include the library. This typically exposes your program to additional
   licensing requirements, including making your application subject to that
   license and therefore open source. See :ref:`licensing_considerations` for
   more.

.. _config_python_distribution_pip_install:

``PythonDistribution.pip_install(args, extra_envs={})``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method runs ``pip install <args>`` with the specified distribution.

``args``
   List of strings defining raw process arguments to pass to ``pip install``.

``extra_envs``
   Optional dict of string key-value pairs constituting extra environment
   variables to set in the invoked ``pip`` process.

Returns a ``list`` of objects representing Python resources installed as
part of the operation. The types of these objects can be ``PythonSourceModule``,
``PythonBytecodeModule``, ``PythonResourceData``, etc.

The returned resources are typically added to a ``FileManifest`` or
``PythonExecutable`` to make them available to a packaged
application.

``PythonDistribution.read_package_root(path, packages)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method discovers resources from a directory on the filesystem.

The specified directory will be scanned for resource files. However,
only specific named *packages* will be found. e.g. if the directory
contains sub-directories ``foo/`` and ``bar``, you must explicitly
state that you want the ``foo`` and/or ``bar`` package to be included
so files from these directories will be read.

This rule is frequently used to pull in packages from local source
directories (e.g. directories containing a ``setup.py`` file). This
rule doesn't involve any packaging tools and is a purely driven by
filesystem walking. It is primitive, yet effective.

This rule has the following arguments:

``path`` (string)
   The filesystem path to the directory to scan.

``packages`` (list of string)
   List of package names to include.

   Filesystem walking will find files in a directory ``<path>/<value>/`` or in
   a file ``<path>/<value>.py``.

Returns a ``list`` of objects representing Python resources found in the virtualenv.
The types of these objects can be ``PythonSourceModule``, ``PythonBytecodeModule``,
``PythonResourceData``, etc.

The returned resources are typically added to a ``FileManifest`` or
``PythonExecutable`` to make them available to a packaged application.

``PythonDistribution.read_virtualenv(path)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method attempts to read Python resources from an already built
virtualenv.

.. important::

   PyOxidizer only supports finding modules and resources
   populated via *traditional* means (e.g. ``pip install`` or ``python setup.py
   install``). If ``.pth`` or similar mechanisms are used for installing modules,
   files may not be discovered properly.

It accepts the following arguments:

``path`` (string)
   The filesystem path to the root of the virtualenv.

   Python modules are typically in a ``lib/pythonX.Y/site-packages`` directory
   (on UNIX) or ``Lib/site-packages`` directory (on Windows) under this path.

Returns a ``list`` of objects representing Python resources found in the virtualenv.
The types of these objects can be ``PythonSourceModule``, ``PythonBytecodeModule``,
``PythonResourceData``, etc.

The returned resources are typically added to a ``FileManifest`` or
``PythonExecutable`` to make them available to a packaged application.

``PythonDistribution.setup_py_install(...)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method runs ``python setup.py install`` against a package at the
specified path.

It accepts the following arguments:

``package_path``
   String filesystem path to directory containing a ``setup.py`` to invoke.

``extra_envs={}``
   Optional dict of string key-value pairs constituting extra environment
   variables to set in the invoked ``python`` process.

``extra_global_arguments=[]``
   Optional list of strings of extra command line arguments to pass to
   ``python setup.py``. These will be added before the ``install``
   argument.

Returns a ``list`` of objects representing Python resources installed
as part of the operation. The types of these objects can be
``PythonSourceModule``, ``PythonBytecodeModule``, ``PythonResourceData``,
etc.

The returned resources are typically added to a ``FileManifest`` or
``PythonExecutable`` to make them available to a packaged application.

.. _config_python_distribution_to_python_executable:

``PythonDistribution.to_python_executable(...)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method constructs a :ref:`config_python_executable` instance. It
essentially says *build an executable embedding Python from this
distribution*.

The accepted arguments are:

``name`` (``str``)
   The name of the application being built. This will be used to construct the
   default filename of the executable.

``config`` (``PythonEmbeddedConfig``)
   The default configuration of the embedded Python interpreter.

   Default is what ``PythonInterpreterConfig()`` returns.

``extension_module_filter`` (``str``)
   The filter to apply to determine which extension modules to add.

   See :ref:`config_python_distribution_extension_modules` for what
   values are accepted and their behavior.

   Default is ``all``.

``preferred_extension_module_variants`` (``dict`` of ``string`` to ``string``)
   Preferred extension module variants to use. See
   See :ref:`config_python_distribution_extension_modules` for behavior.

   Default is ``None``, which will use the first variant.

``include_sources`` (``bool``)
   Boolean to control whether sources of Python modules are added in addition
   to bytecode.

   Default is ``True``.

``include_resources`` (``bool``)
   Boolean to control whether non-module resource data from the distribution is
   added.

   Default is ``False``.

``include_test`` (``bool``)
   Boolean to control whether test-specific objects are included.

   Default is ``False``.

.. _config_python_resources:

Python Resources
================

At run-time, Python interpreters need to consult *resources* like Python
module source and bytecode as well as resource/data files. We refer to all
of these as *Python Resources*.

Configuration files represent *Python Resources* via the types
:ref:`config_python_source_module`, :ref:`config_python_bytecode_module`,
:ref:`config_python_resources_data`, and :ref:`config_python_extension_module`.

These are described in detail in the following sections.

.. _config_python_source_module:

``PythonSourceModule``
----------------------

This type represents Python source modules, agnostic of location.

Each instance has the following attributes:

``name`` (string)
   Fully qualified name of the module. e.g. ``foo.bar``.

``is_package`` (bool)
   Whether this module is also a Python package (or sub-package).

Instances cannot be manually constructed.

.. _config_python_bytecode_module:

``PythonBytecodeModule``
------------------------

This type represents a Python module defined through bytecode.

Each instance has the following attributes:

``name`` (string)
   Fully qualified name of the module. e.g. ``foo.bar``

``optimize_level`` (int)
   Optimization level of compiled bytecode. Must be the value
   ``0``, ``1``, or ``2``.

``is_package`` (bool)
   Whether the module is also a Python package (or sub-package).

.. _config_python_resources_data:

``PythonResourcesData``
-----------------------

This type represents Python resource data. Resource data is a named
blob associated with a Python package. It is typically accessed using
the ``importlib.resources`` API.

Each instance has the following attributes:

``package`` (string)
   Python package this resource is associated with.

``name`` (string)
   Name of this resource.

.. _config_python_extension_module:

``PythonExtensionModule``
-------------------------

This type represents a compiled Python extension module.

Each instance has the following attributes:

``name`` (string)
   Unique name of the module being provided.

Python Interpreter Configuration
================================

A Python interpreter has settings to control how it runs. Configuration
files represent these settings through the
:ref:`config_python_interpreter_config` type.

.. _config_python_interpreter_config:

``PythonInterpreterConfig(...)```
---------------------------------

This type configures the default behavior of the embedded Python interpreter.

Embedded Python interpreters are configured and instantiated using a
``pyembed::PythonConfig`` data structure. The ``pyembed`` crate defines a
default instance of this data structure with parameters defined by the settings
in this type.

.. note::

   If you are writing custom Rust code and constructing a custom
   ``pyembed::PythonConfig`` instance and don't use the default instance, this
   config type is not relevant to you and can be omitted from your config
   file.

The following arguments can be defined to control the default ``PythonConfig``
behavior:

``bytes_warning`` (int)
   Controls the value of
   `Py_BytesWarningFlag <https://docs.python.org/3/c-api/init.html#c.Py_BytesWarningFlag>`_.

   Default is ``0``.

``filesystem_importer`` (bool)
   Controls whether to enable Python's filesystem based importer. Enabling
   this importer allows Python modules to be imported from the filesystem.

   Default is ``False`` (since PyOxidizer prefers embedding Python modules in
   binaries).

``ignore_environment`` (bool)
   Controls the value of
   `Py_IgnoreEnvironmentFlag <https://docs.python.org/3/c-api/init.html#c.Py_IgnoreEnvironmentFlag>`_.

   This is likely wanted for embedded applications that don't behave like
   ``python`` executables.

   Default is ``True``.

``inspect`` (bool)
   Controls the value of
   `Py_InspectFlag <https://docs.python.org/3/c-api/init.html#c.Py_InspectFlag>`_.

   Default is ``False``.

``interactive`` (bool)
   Controls the value of
   `Py_InteractiveFlag <https://docs.python.org/3/c-api/init.html#c.Py_InspectFlag>`_.

   Default is ``False``.

``isolated`` (bool)
   Controls the value of
   `Py_IsolatedFlag <https://docs.python.org/3/c-api/init.html#c.Py_IsolatedFlag>`_.

``legacy_windows_fs_encoding`` (bool)
   Controls the value of
   `Py_LegacyWindowsFSEncodingFlag <https://docs.python.org/3/c-api/init.html#c.Py_LegacyWindowsFSEncodingFlag>`_.

   Only affects Windows.

   Default is ``False``.

``legacy_windows_stdio`` (bool)
   Controls the value of
   `Py_LegacyWindowsStdioFlag <https://docs.python.org/3/c-api/init.html#c.Py_LegacyWindowsStdioFlag>`_.

   Only affects Windows.

   Default is ``False``.

``optimize_level`` (bool)
   Controls the value of
   `Py_OptimizeFlag <https://docs.python.org/3/c-api/init.html#c.Py_OptimizeFlag>`_.

   Default is ``0``, which is the Python default. Only the values ``0``, ``1``,
   and ``2`` are accepted.

   This setting is only relevant if ``dont_write_bytecode`` is ``false`` and Python
   modules are being imported from the filesystem.

``parser_debug`` (bool)
   Controls the value of
   `Py_DebugFlag <https://docs.python.org/3/c-api/init.html#c.Py_DebugFlag>`_.

   Default is ``False``.

``quiet`` (bool)
   Controls the value of
   `Py_QuietFlag <https://docs.python.org/3/c-api/init.html#c.Py_QuietFlag>`_.

``raw_allocator`` (string)
   Which memory allocator to use for the ``PYMEM_DOMAIN_RAW`` allocator.

   This controls the lowest level memory allocator used by Python. All Python
   memory allocations use memory allocated by this allocator (higher-level
   allocators call into this pool to allocate large blocks then allocate
   memory out of those blocks instead of using the *raw* memory allocator).

   Values can be ``jemalloc``, ``rust``, or ``system``.

   ``jemalloc`` will have Python use the jemalloc allocator directly.

   ``rust`` will use Rust's global allocator (whatever that may be).

   ``system`` will use the default allocator functions exposed to the binary
   (``malloc()``, ``free()``, etc).

   The ``jemalloc`` allocator requires the ``jemalloc-sys`` crate to be
   available. A run-time error will occur if ``jemalloc`` is configured but this
   allocator isn't available.

   **Important**: the ``rust`` crate is not recommended because it introduces
   performance overhead.

   Default is ``jemalloc`` on non-Windows targets and ``system`` on Windows.
   (The ``jemalloc-sys`` crate doesn't work on Windows MSVC targets.)

``run_eval`` (string)
   Will cause the interpreter to evaluate a Python code string defined by this
   value after the interpreter initializes.

   An example value would be ``import mymodule; mymodule.main()``.

``run_file`` (string)
   Will cause the interpreter to evaluate a file at the specified filename.

   The filename is resolved at run-time using whatever mechanisms the Python
   interpreter applies. i.e. this is little different from running
   ``python <path>``.

``run_module`` (string)
   The Python interpreter will load a Python module with this value's name
   as the ``__main__`` module and then execute that module.

``run_noop`` (bool)
   Instructs the Python interpreter to do nothing after initialization.

``run_repl`` (bool)
   The Python interpreter will launch an interactive Python REPL connected to
   stdio. This is similar to the default behavior of running a ``python``
   executable without any arguments.

``site_import`` (bool)
   Controls the inverse value of
   `Py_NoSiteFlag <https://docs.python.org/3/c-api/init.html#c.Py_NoSiteFlag>`_.

   The ``site`` module is typically not needed for standalone Python applications.

   Default is ``False``.

``stdio_encoding`` (string)
   Defines the encoding and error handling mode for Python's standard I/O
   streams (``sys.stdout``, etc). Values are of the form ``encoding:error`` e.g.
   ``utf-8:ignore`` or ``latin1-strict``.

   If defined, the ``Py_SetStandardStreamEncoding()`` function is called during
   Python interpreter initialization. If not, the Python defaults are used.

``sys_frozen`` (bool)
   Controls whether to set the ``sys.frozen`` attribute to ``True``. If
   ``false``, ``sys.frozen`` is not set.

   Default is ``False``.

``sys_meipass`` (bool)
   Controls whether to set the ``sys._MEIPASS`` attribute to the path of
   the executable.

   Setting this and ``sys_frozen`` to ``true`` will emulate the
   `behavior of PyInstaller <https://pyinstaller.readthedocs.io/en/v3.3.1/runtime-information.html>`_
   and could possibly help self-contained applications that are aware of
   PyInstaller also work with PyOxidizer.

   Default is ``False``.

``sys_paths`` (array of strings)
   Defines filesystem paths to be added to ``sys.path``.

   Setting this value will imply ``filesystem_importer = true``.

   The special token ``$ORIGIN`` in values will be expanded to the absolute
   path of the directory of the executable at run-time. For example,
   if the executable is ``/opt/my-application/pyapp``, ``$ORIGIN`` will
   expand to ``/opt/my-application`` and the value ``$ORIGIN/lib`` will
   expand to ``/opt/my-application/lib``.

   If defined in multiple sections, new values completely overwrite old
   values (values are not merged).

   Default is an empty array (``[]``).

.. _config_terminfo_resolution:

``terminfo_resolution`` (string)
   How the terminal information database (``terminfo``) should be configured.

   See :ref:`terminfo_database` for more about terminal databases.

   The value ``dynamic`` (the default) looks at the currently running
   operating system and attempts to do something reasonable. For example, on
   Debian based distributions, it will look for the ``terminfo`` database in
   ``/etc/terminfo``, ``/lib/terminfo``, and ``/usr/share/terminfo``, which is
   how Debian configures ``ncurses`` to behave normally. Similar behavior exists
   for other recognized operating systems. If the operating system is unknown,
   PyOxidizer falls back to looking for the ``terminfo`` database in well-known
   directories that often contain the database (like ``/usr/share/terminfo``).

   The value ``none`` indicates that no configuration of the ``terminfo``
   database path should be performed. This is useful for applications that
   don't interact with terminals. Using ``none`` can prevent some filesystem
   I/O at application startup.

   The value ``static`` indicates that a static path should be used for the
   path to the ``terminfo`` database. That path should be provided by the
   ``terminfo_dirs`` configuration option.

   ``terminfo`` is not used on Windows and this setting is ignored on that
   platform.

``terminfo_dirs``
   Path to the ``terminfo`` database. See the above documentation for
   ``terminfo_resolution`` for more on the ``terminfo`` database.

   This value consists of a ``:`` delimited list of filesystem paths that
   ``ncurses`` should be configured to use. This value will be used to
   populate the ``TERMINFO_DIRS`` environment variable at application run time.

``unbuffered_stdio`` (bool)
   Controls the value of
   `Py_UnbufferedStdioFlag <https://docs.python.org/3/c-api/init.html#c.Py_UnbufferedStdioFlag>`_.

   Setting this makes the standard I/O streams unbuffered.

   Default is ``False``.

``use_hash_seed`` (bool)
   Controls the value of
   `Py_HashRandomizationFlag <https://docs.python.org/3/c-api/init.html#c.Py_HashRandomizationFlag>`_.

   Default is ``False``.

``user_site_directory`` (bool)
   Controls the inverse value of
   `Py_NoUserSiteDirectory <https://docs.python.org/3/c-api/init.html#c.Py_NoUserSiteDirectory>`_.

   Default is ``False``.

``write_bytecode`` (bool)
   Controls the inverse value of
   `Py_DontWriteBytecodeFlag <https://docs.python.org/3/c-api/init.html#c.Py_DontWriteBytecodeFlag>`_.

   This is only relevant if the interpreter is configured to import modules
   from the filesystem.

   Default is ``False``.

``write_modules_directory_env`` (string)
   Environment variable that defines a directory where ``modules-<UUID>`` files
   containing a ``\n`` delimited list of loaded Python modules (from ``sys.modules``)
   will be written upon interpreter shutdown.

   If this setting is not defined or if the environment variable specified by its
   value is not present at run-time, no special behavior will occur. Otherwise,
   the environment variable's value is interpreted as a directory, that directory
   and any of its parents will be created, and a ``modules-<UUID>`` file will
   be written to the directory.

   This setting is useful for determining which Python modules are loaded when
   running Python code.

.. _config_python_binaries:

Python Binaries
===============

Binaries containing an embedded Python interpreter can be defined by
configuration files. They are defined via the :ref:`config_python_executable`
type. In addition, the :ref:`config_python_embedded_data` type defines the raw
resources that constitute an embedded Python interpreter.

.. _config_python_embedded_data:

``PythonEmbeddedData``
----------------------

The ``PythonEmbeddedData`` type represents resources embedded within a binary
to provide a Python interpreter. The various resources tracked by this type are
consumed by the ``pyembed`` at build and run time. Various tracked resources
include:

* A link library providing the Python interpreter symbols.
* A :ref:`config_python_interpreter_config` defining a default Python interpreter
  configuration.
* Python module and resource data to be embedded in the binary.

Instances of this type are constructed by transforming a type representing
a Python binary. e.g. :ref:`config_python_executable_to_embedded_data`.

If this type is returned by a target function, its build action will write
out files that represent the various resources encapsulated by this type. There
is no run action associated with this type.

.. _config_python_executable:

``PythonExecutable``
--------------------

The ``PythonExecutable`` type represents an executable file containing
the Python interpreter, Python resources to make available to the interpreter,
and a default run-time configuration for that interpreter.

Instances are constructed from ``PythonDistribution`` instances using
:ref:`config_python_distribution_to_python_executable`.

.. _config_python_executable_add_module_source:

``PythonExecutable.add_module_source(module)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method registers a Python source module with a ``PythonExecutable``
instance. The argument must be a ``PythonSourceModule`` instance.

If called multiple times for the same module, the last write wins.

.. _config_python_executable_add_module_bytecode:

``PythonExecutable.add_module_bytecode(module, optimize_level=0)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method registers a Python module bytecode with a
``PythonExecutable`` instance. The first argument must be a
``PythonSourceModule`` instance and the 2nd argument the value ``0``, ``1``,
or ``2``.

Only one level of bytecode can be registered per named module. If called
multiple times for the same module, the last write wins.

.. _config_python_executable_add_resource_data:

``PythonExecutable.add_resource_data(resource)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method adds a ``PythonResourceData`` instance to the
``PythonExecutable`` instance, making that resource available
via in-memory access.

If multiple resources sharing the same ``(package, name)`` pair are added,
the last added one is used.

.. _config_python_executable_add_extension_module:

``PythonExecutable.add_extension_module(module)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method registers a ``PythonExtensionModule`` instance with a
``PythonExecutable`` instance. The extension module will be statically
linked into the binary produced from the ``PythonExecutable``
instance.

If multiple extension modules with the same name are added, the last
added one is used.

.. _config_python_executable_add_python_resource:

``PythonExecutable.add_python_resource(...)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method registers a Python resource of various types. It accepts a
``resource`` argument which can be a ``PythonSourceModule``,
``PythonBytecodeModule``, ``PythonResourceData``, or ``PythonExtensionModule``
and registers that resource with this instance. This method is a glorified
proxy to the appropriate ``add_*`` method.

The following arguments are accepted:

``resource``
   The resource to add to the embedded Python environment.

``add_source_module`` (bool)
   When the resource is a ``PythonSourceModule``, this flag determines
   whether to add the source for that resource.

   Default is ``True``.

``add_bytecode_module`` (bool)
   When the resource is a ``PythonSourceModule``, this flag determines
   whether to add the bytecode for that module source.

   Default is ``True``.

``optimize_level`` (int)
   Bytecode optimization level when compiling bytecode.

.. _config_python_executable_add_python_resources:

``PythonExecutable.add_python_resources(...)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method registers an iterable of Python resources of various types.
This method is identical to ``PythonExecutable.add_python_resource()``
except the first argument is an iterable of resources. All other arguments
are identical.

.. _config_python_executable_filter_from_files:

``PythonExecutable.filter_from_files(files=[], glob_patterns=[])``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method filters all embedded resources (source modules, bytecode modules,
and resource names) currently present on the instance through a set of
resource names resolved from files.

This method accepts the following arguments:

``files`` (array of string)
   List of filesystem paths to files containing resource names. The file
   must be valid UTF-8 and consist of a ``\n`` delimited list of resource
   names. Empty lines and lines beginning with ``#`` are ignored.

``glob_files`` (array of string)
   List of glob matching patterns of filter files to read. ``*`` denotes
   all files in a directory. ``**`` denotes recursive directories. This
   uses the Rust ``glob`` crate under the hood and the documentation for that
   crate contains more pattern matching info.

   The files read by this argument must be the same format as documented
   by the ``files`` argument.

All defined files are first read and the resource names encountered are
unioned into a set. This set is then used to filter entities currently
registered with the instance.

.. _config_python_executable_to_embedded_data:

``PythonExecutable.to_embedded_data()``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Obtains a :ref:`config_python_embedded_data` instance representing resources
to be embedded in a binary which are then used by the ``pyembed`` Rust crate
to instantiate and run a Python interpreter.

See the :ref:`config_python_embedded_data` type documentation for more.

Interacting With the Filesystem
===============================

.. _config_file_manifest:

``FileManifest()``
------------------

The ``FileManifest`` type represents a set of files and their content.

``FileManifest`` instances are used to represent things like the final
filesystem layout of an installed application.

Conceptually, a ``FileManifest`` is a dict mapping relative paths to
file content.

.. _config_file_manifest_add_manifest:

``FileManifest.add_manifest(manifest)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method overlays another ``FileManifest`` on this one. If the other
manifest provides a path already in this manifest, its content will be
replaced by what is in the other manifest.

``FileManifest.add_python_resource(prefix, value)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method adds a Python resource to a ``FileManifest`` instance in
a specified directory prefix. A *Python resource* here can be a
``PythonSourceModule``, ``PythonBytecodeModule``, ``PythonResourceData``,
or ``PythonExtensionModule``.

This method can be used to place the Python resources derived from another
type or action in the filesystem next to an application binary.

``FileManifest.add_python_resources(prefix, values)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method adds an iterable of Python resources to a ``FileManifest``
instance in a specified directory prefix. This is effectively a wrapper
for ``for value in values: self.add_python_resource(prefix, value)``.

For example, to place the Python distribution's standard library Python
source modules in a directory named ``lib``::

   m = FileManifest()
   dist = default_python_distribution()
   m.add_python_resources(dist.source_modules())

``FileManifest.install(path, replace=True)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This method writes the content of the ``FileManifest`` to a directory
specified by ``path``. The path is evaluated relative to the path
specified by ``BUILD_PATH``.

If ``replace`` is True (the default), the destination directory will
be deleted and the final state of the destination directory should
exactly match the state of the ``FileManifest``.

.. _config_glob:

``glob(include, exclude=None, strip_prefix=None)``
--------------------------------------------------

The ``glob()`` function resolves file patterns to a ``FileManifest``.

``include`` is a ``list`` of ``str`` containing file patterns that will be
matched using the ``glob`` Rust crate. If patterns begin with ``/`` or
look like a filesystem absolute path, they are absolute. Otherwise they are
evaluated relative to the directory of the current config file.

``exclude`` is an optional ``list`` of ``str`` and is used to exclude files
from the result. All patterns in ``include`` are evaluated before ``exclude``.

``strip_prefix`` is an optional ``str`` to strip from the beginning of
matched files. ``strip_prefix`` is stripped after ``include`` and ``exclude``
are processed.

Returns a ``FileManifest``.
