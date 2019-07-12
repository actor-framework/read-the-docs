.. _system-config:

Configuring Actor Applications
==============================

CAF configures applications at startup using an ``actor_system_config`` or a user-defined subclass of that type. The config objects allow users to add custom types, to load modules, and to fine-tune the behavior of loaded modules with command line options or configuration files .

The following code example is a minimal CAF application with a middleman  but without any custom configuration options.

::

   void caf_main(actor_system& system) {
     // ...
   }
   CAF_MAIN(io::middleman)

The compiler expands this example code to the following.

::

   void caf_main(actor_system& system) {
     // ...
   }
   int main(int argc, char** argv) {
     return exec_main<io::middleman>(caf_main, argc, argv);
   }

The function ``exec_main`` creates a config object, loads all modules requested in ``CAF_MAIN`` and then calls ``caf_main``. A minimal implementation for ``main`` performing all these steps manually is shown in the next example for the sake of completeness.

::

   int main(int argc, char** argv) {
     actor_system_config cfg;
     // read CLI options
     cfg.parse(argc, argv);
     // return immediately if a help text was printed
     if (cfg.cli_helptext_printed)
       return 0;
     // load modules
     cfg.load<io::middleman>();
     // create actor system and call caf_main
     actor_system system{cfg};
     caf_main(system);
   }

However, setting up config objects by hand is usually not necessary. CAF automatically selects user-defined subclasses of ``actor_system_config`` if ``caf_main`` takes a second parameter by reference, as shown in the minimal example below.

::

   class my_config : public actor_system_config {
   public:
     my_config() {
       // ...
     }
   };

   void caf_main(actor_system& system, const my_config& cfg) {
     // ...
   }

   CAF_MAIN()

Users can perform additional initialization, add custom program options, etc. simply by implementing a default constructor.

.. _system-config-module:

Loading Modules
---------------

The simplest way to load modules is to use the macro ``CAF_MAIN`` and to pass a list of all requested modules, as shown below.

::

   void caf_main(actor_system& system) {
     // ...
   }
   CAF_MAIN(mod1, mod2, ...)

Alternatively, users can load modules in user-defined config classes.

::

   class my_config : public actor_system_config {
   public:
     my_config() {
       load<mod1>();
       load<mod2>();
       // ...
     }
   };

The third option is to simply call ``x.load<mod1>()`` on a config object *before* initializing an actor system with it.

.. _system-config-options:

Command Line Options and INI Configuration Files
------------------------------------------------

CAF organizes program options in categories and parses CLI arguments as well as INI files. CLI arguments override values in the INI file which override hard-coded defaults. Users can add any number of custom program options by implementing a subtype of ``actor_system_config``. The example below adds three options to the “global” category.

.. code-block:: C++

   public:
     uint16_t port = 0;
     std::string host = "localhost";
     bool server_mode = false;
   
     config() {
       opt_group{custom_options_, "global"}
       .add(port, "port,p", "set port")
       .add(host, "host,H", "set host (ignored in server mode)")
       .add(server_mode, "server-mode,s", "enable server mode");
     }
   };
   

We create a new “global” category in ``custom_options_}``. Each following call to ``add`` then appends individual options to the category. The first argument to ``add`` is the associated variable. The second argument is the name for the parameter, optionally suffixed with a comma-separated single-character short name. The short name is only considered for CLI parsing and allows users to abbreviate commonly used option names. The third and final argument to ``add`` is a help text.

The custom ``config`` class allows end users to set the port for the application to 42 with either ``--port=42`` (long name) or ``-p 42`` (short name). The long option name is prefixed by the category when using a different category than “global”. For example, adding the port option to the category “foo” means end users have to type ``--foo.port=42`` when using the long name. Short names are unaffected by the category, but have to be unique.

Boolean options do not require arguments. The member variable ``server_mode`` is set to ``true`` if the command line contains either ``--server-mode`` or ``-s``.

The example uses member variables for capturing user-provided settings for simplicity. However, this is not required. For example, ``add<bool>(...)`` allows omitting the first argument entirely. All values of the configuration are accessible with ``get_or``. Note that all global options can omit the ``"global."`` prefix.

CAF adds the program options “help” (with short names ``-h`` and ``-?``) as well as “long-help” to the “global” category.

The default name for the INI file is ``caf-application.ini``. Users can change the file name and path by passing ``--config-file=<path>`` on the command line.

INI files are organized in categories. No value is allowed outside of a category (no implicit “global” category). The parses uses the following syntax:

+-------------------------+-----------------------------+
| ``key=true``            | is a boolean                |
+-------------------------+-----------------------------+
| ``key=1``               | is an integer               |
+-------------------------+-----------------------------+
| ``key=1.0``             | is an floating point number |
+-------------------------+-----------------------------+
| ``key=1ms``             | is an timespan              |
+-------------------------+-----------------------------+
| ``key='foo'``           | is an atom                  |
+-------------------------+-----------------------------+
| ``key="foo"``           | is a string                 |
+-------------------------+-----------------------------+
| ``key=[0, 1, ...]``     | is as a list                |
+-------------------------+-----------------------------+
| ``key={a=1, b=2, ...}`` | is a dictionary (map)       |
+-------------------------+-----------------------------+

The following example INI file lists all standard options in CAF and their default value. Note that some options such as ``scheduler.max-threads`` are usually detected at runtime and thus have no hard-coded default.

.. code-block:: ini

   ; This file shows all possible parameters with defaults.
   ; Values enclosed in <> are detected at runtime unless defined by the user.
   
   ; when using the default scheduler
   [scheduler]
   ; accepted alternative: 'sharing'
   policy='stealing'
   ; configures whether the scheduler generates profiling output
   enable-profiling=false
   ; forces a fixed number of threads if set
   max-threads=<number of cores>
   ; maximum number of messages actors can consume in one run
   max-throughput=<infinite>
   ; measurement resolution in milliseconds (only if profiling is enabled)
   profiling-resolution=100ms
   ; output file for profiler data (only if profiling is enabled)
   profiling-output-file="/dev/null"
   
   ; when using 'stealing' as scheduler policy
   [work-stealing]
   ; number of zero-sleep-interval polling attempts
   aggressive-poll-attempts=100
   ; frequency of steal attempts during aggressive polling
   aggressive-steal-interval=10
   ; number of moderately aggressive polling attempts
   moderate-poll-attempts=500
   ; frequency of steal attempts during moderate polling
   moderate-steal-interval=5
   ; sleep interval between poll attempts
   moderate-sleep-duration=50us
   ; frequency of steal attempts during relaxed polling
   relaxed-steal-interval=1
   ; sleep interval between poll attempts
   relaxed-sleep-duration=10ms
   
   ; when loading io::middleman
   [middleman]
   ; configures whether MMs try to span a full mesh
   enable-automatic-connections=false
   ; application identifier of this node, prevents connection to other CAF
   ; instances with different identifier
   app-identifier=""
   ; maximum number of consecutive I/O reads per broker
   max-consecutive-reads=50
   ; heartbeat message interval in ms (0 disables heartbeating)
   heartbeat-interval=0ms
   ; configures whether the MM attaches its internal utility actors to the
   ; scheduler instead of dedicating individual threads (needed only for
   ; deterministic testing)
   attach-utility-actors=false
   ; configures whether the MM starts a background thread for I/O activity,
   ; setting this to true allows fully deterministic execution in unit test and
   ; requires the user to trigger I/O manually
   manual-multiplexing=false
   ; disables communication via TCP
   disable-tcp=false
   ; enable communication via UDP
   enable-udp=false
   ; configures how many background workers are spawned for deserialization,
   ; by default CAF uses 1-4 workers depending on the number of cores
   workers=<min(3, number of cores / 4) + 1>
   
   ; when compiling with logging enabled
   [logger]
   ; file name template for output log file files (empty string disables logging)
   file-name="actor_log_[PID]_[TIMESTAMP]_[NODE].log"
   ; format for rendering individual log file entries
   file-format="%r %c %p %a %t %C %M %F:%L %m%n"
   ; configures the minimum severity of messages that are written to the log file
   ; (quiet|error|warning|info|debug|trace)
   file-verbosity='trace'
   ; mode for console log output generation (none|colored|uncolored)
   console='none'
   ; format for printing individual log entries to the console
   console-format="%m"
   ; configures the minimum severity of messages that are written to the console
   ; (quiet|error|warning|info|debug|trace)
   console-verbosity='trace'
   ; excludes listed components from logging (list of atoms)
   component-blacklist=[]

.. _add-custom-message-type:

Adding Custom Message Types
---------------------------

CAF requires serialization support for all of its message types . However, CAF also needs a mapping of unique type names to user-defined types at runtime. This is required to deserialize arbitrary messages from the network.

As an introductory example, we (again) use the following POD type ``foo``.

.. code-block:: C++

   struct foo {
     std::vector<int> a;
     int b;
   };

To make ``foo`` serializable, we make it inspectable :

.. code-block:: C++

   template <class Inspector>
   typename Inspector::result_type inspect(Inspector& f, foo& x) {
     return f(meta::type_name("foo"), x.a, x.b);
   }
   

Finally, we give ``foo`` a platform-neutral name and add it to the list of serializable types by using a custom config class.

.. code-block:: C++

   class config : public actor_system_config {
   public:
     config() {
       add_message_type<foo>("foo");
     }
   };
   
   void caf_main(actor_system& system, const config&) {

.. _adding-custom-error-types:

Adding Custom Error Types
-------------------------

Adding a custom error type to the system is a convenience feature to allow improve the string representation. Error types can be added by implementing a render function and passing it to ``add_error_category``, as shown in .

.. _add-custom-actor-type:

Adding Custom Actor Types 
--------------------------

Adding actor types to the configuration allows users to spawn actors by their name. In particular, this enables spawning of actors on a different node . For our example configuration, we consider the following simple ``calculator`` actor.

.. code-block:: C++

   using add_atom = atom_constant<atom("add")>;
   using sub_atom = atom_constant<atom("sub")>;
   
   using calculator = typed_actor<replies_to<add_atom, int, int>::with<int>,
                                  replies_to<sub_atom, int, int>::with<int>>;
   
   calculator::behavior_type calculator_fun(calculator::pointer self) {

Adding the calculator actor type to our config is achieved by calling ``add_actor_type<T>``. Note that adding an actor type in this way implicitly calls ``add_message_type<T>`` for typed actors . This makes our ``calculator`` actor type serializable and also enables remote nodes to spawn calculators anywhere in the distributed actor system (assuming all nodes use the same config).

.. code-block:: C++

   struct config : actor_system_config {
     config() {
       add_actor_type("calculator", calculator_fun);
     }

Our final example illustrates how to spawn a ``calculator`` locally by using its type name. Because the dynamic type name lookup can fail and the construction arguments passed as message can mismatch, this version of ``spawn`` returns ``expected<T>``.

::

   auto x = system.spawn<calculator>("calculator", make_message());
   if (! x) {
     std::cerr << "*** unable to spawn calculator: "
               << system.render(x.error()) << std::endl;
     return;
   }
   calculator c = std::move(*x);

Adding dynamically typed actors to the config is achieved in the same way. When spawning a dynamically typed actor in this way, the template parameter is simply ``actor``. For example, spawning an actor "foo" which requires one string is created with:

::

   auto worker = system.spawn<actor>("foo", make_message("bar"));

Because constructor (or function) arguments for spawning the actor are stored in a ``message``, only actors with appropriate input types are allowed. For example, pointer types are illegal. Hence users need to replace C-strings with ``std::string``.

.. _log-output:

Log Output
----------

Logging is disabled in CAF per default. It can be enabled by setting the ``--with-log-level=`` option of the ``configure`` script to one of “error”, “warning”, “info”, “debug”, or “trace” (from least output to most). Alternatively, setting the CMake variable ``CAF_LOG_LEVEL`` to 0, 1, 2, 3, or 4 (from least output to most) has the same effect.

All logger-related configuration options listed here and in are silently ignored if logging is disabled.

.. _log-output-file-name:

File Name
~~~~~~~~~

The output file is generated from the template configured by ``logger-file-name``. This template supports the following variables.

+-----------------+--------------------------------+
| **Variable**    | **Output**                     |
+=================+================================+
| ``[PID]``       | The OS-specific process ID.    |
+-----------------+--------------------------------+
| ``[TIMESTAMP]`` | The UNIX timestamp on startup. |
+-----------------+--------------------------------+
| ``[NODE]``      | The node ID of the CAF system. |
+-----------------+--------------------------------+

.. _log-output-console:

Console
~~~~~~~

Console output is disabled per default. Setting ``logger-console`` to either ``"uncolored"`` or ``"colored"`` prints log events to ``std::clog``. Using the ``"colored"`` option will print the log events in different colors depending on the severity level.

.. _log-output-format-strings:

Format Strings
~~~~~~~~~~~~~~

CAF uses log4j-like format strings for configuring printing of individual events via ``logger-file-format`` and ``logger-console-format``. Note that format modifiers are not supported at the moment. The recognized field identifiers are:

+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| **Character** | **Output**                                                                                                                       |
+===============+==================================================================================================================================+
| ``c``         | The category/component. This name is defined by the macro ``CAF_LOG_COMPONENT``. Set this macro before including any CAF header. |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``C``         | The full qualifier of the current function. For example, the qualifier of ``void ns::foo::bar()`` is printed as ``ns.foo``.      |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``d``         | The date in ISO 8601 format, i.e., ``"YYYY-MM-DD hh:mm:ss"``.                                                                    |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``F``         | The file name.                                                                                                                   |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``L``         | The line number.                                                                                                                 |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``m``         | The user-defined log message.                                                                                                    |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``M``         | The name of the current function. For example, the name of ``void ns::foo::bar()`` is printed as ``bar``.                        |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``n``         | A newline.                                                                                                                       |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``p``         | The priority (severity level).                                                                                                   |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``r``         | Elapsed time since starting the application in milliseconds.                                                                     |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``t``         | ID of the current thread.                                                                                                        |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``a``         | ID of the current actor (or “actor0” when not logging inside an actor).                                                          |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+
| ``%``         | A single percent sign.                                                                                                           |
+---------------+----------------------------------------------------------------------------------------------------------------------------------+

.. _log-output-filtering:

Filtering
~~~~~~~~~

The two configuration options ``logger-component-filter`` and ``logger-verbosity`` reduce the amount of generated log events. The former is a list of excluded component names and the latter can increase the reported severity level (but not decrease it beyond the level defined at compile time).
