.. _system-config:

Configuring Actor Applications
==============================

CAF configures applications at startup using an ``actor_system_config`` or a user-defined subclass of that type. The config objects allow users to add custom types, to load modules, and to fine-tune the behavior of loaded modules with command line options or configuration files (see :ref:`system-config-options`).

The following code example is a minimal CAF application with a middleman (see :ref:`middleman`) but without any custom configuration options.

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

::

    class config : public actor_system_config {
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

The line ``opt_group{custom_options_, "global"}`` adds the “global” category to the config parser. The following calls to ``add`` then append individual options to the category. The first argument to ``add`` is the associated variable. The second argument is the name for the parameter, optionally suffixed with a comma-separated single-character short name. The short name is only considered for CLI parsing and allows users to abbreviate commonly used option names. The third and final argument to ``add`` is a help text.

The custom ``config`` class allows end users to set the port for the application to 42 with either ``--port=42`` (long name) or ``-p 42`` (short name). The long option name is prefixed by the category when using a different category than “global”. For example, adding the port option to the category “foo” means end users have to type ``--foo.port=42`` when using the long name. Short names are unaffected by the category, but have to be unique.

Boolean options do not require arguments. The member variable ``server_mode`` is set to ``true`` if the command line contains either ``--server-mode`` or ``-s``.

CAF prefixes all of its default CLI options with ``caf#``, except for “help” (``--help``, ``-h``, or ``-?``). The default name for the INI file is ``caf-application.ini``. Users can change the file name and path by passing ``--caf#config-file=<path>`` on the command line.

INI files are organized in categories. No value is allowed outside of a category (no implicit “global” category). CAF reads ``true`` and ``false`` as boolean, numbers as (signed) integers or ``double``, ``"``-enclosed characters as strings, and ``'``-enclosed characters as atoms (see :ref:`atom`). The following example INI file lists all standard options in CAF and their default value. Note that some options such as ``scheduler.max-threads`` are usually detected at runtime and thus have no hard-coded default.

::

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
    profiling-ms-resolution=100
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
    ; sleep interval in microseconds between poll attempts
    moderate-sleep-duration=50
    ; frequency of steal attempts during relaxed polling
    relaxed-steal-interval=1
    ; sleep interval in microseconds between poll attempts
    relaxed-sleep-duration=10000

    ; when loading io::middleman
    [middleman]
    ; configures whether MMs try to span a full mesh
    enable-automatic-connections=false
    ; accepted alternative: 'asio' (only when compiling CAF with ASIO)
    network-backend='default'
    ; application identifier of this node
    app-identifier=""
    ; maximum number of consecutive I/O reads per broker
    max-consecutive-reads=50
    ; heartbeat message interval in ms (0 disables heartbeating)
    heartbeat-interval=0

.. _add-custom-message-type:

Adding Custom Message Types
---------------------------

CAF requires serialization support for all of its message types (see :ref:`type-inspection`). However, CAF also needs a mapping of unique type names to user-defined types at runtime. This is required to deserialize arbitrary messages from the network.

As an introductory example, we (again) use the following POD type ``foo``.

::

    struct foo {
      std::vector<int> a;
      int b;
    };

To make ``foo`` serializable, we make it inspectable (see :ref:`type-inspection`):

::

    template <class Inspector>
    typename Inspector::result_type inspect(Inspector& f, foo& x) {
      return f(meta::type_name("foo"), x.a, x.b);
    }

Finally, we give ``foo`` a platform-neutral name and add it to the list of serializable types by using a custom config class.

::

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

Adding a custom error type to the system is a convenience feature to allow improve the string representation. Error types can be added by implementing a render function and passing it to ``add_error_category``, as shown in :ref:`custom-error`.

.. _add-custom-actor-type:

Adding Custom Actor Types  :sup:`experimental` 
----------------------------------------------

Adding actor types to the configuration allows users to spawn actors by their name. In particular, this enables spawning of actors on a different node (see :ref:`remote-spawn`). For our example configuration, we consider the following simple ``calculator`` actor.

::

    using add_atom = atom_constant<atom("add")>;
    using sub_atom = atom_constant<atom("sub")>;

    using calculator = typed_actor<replies_to<add_atom, int, int>::with<int>,
                                   replies_to<sub_atom, int, int>::with<int>>;

    calculator::behavior_type calculator_fun(calculator::pointer self) {

Adding the calculator actor type to our config is achieved by calling ``add_actor_type<T>``. Note that adding an actor type in this way implicitly calls ``add_message_type<T>`` for typed actors (see :ref:`add-custom-message-type`). This makes our ``calculator`` actor type serializable and also enables remote nodes to spawn calculators anywhere in the distributed actor system (assuming all nodes use the same config).

::

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

Adding dynamically typed actors to the config is achieved in the same way. When spawning a dynamically typed actor in this way, the template parameter is simply ``actor``. For example, spawning an actor “foo” which requires one string is created with ``system.spawn<actor>("foo", make_message("bar"))``.

Because constructor (or function) arguments for spawning the actor are stored in a ``message``, only actors with appropriate input types are allowed. For example, ``const char*`` arguments—or any other pointer type—are not allowed and must be replaced by ``std::string``.
