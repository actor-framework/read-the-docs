.. _system-config:

Configuring Actor Applications
==============================

CAFconfigures applications at startup using an ``actor_system_config`` or a user-defined subclass of that type. The config objects allow users to add custom types, to load modules, and to fine-tune the behavior of loaded modules with command line options or configuration files (see :ref:`system-config-options`).

The following code example is a minimal CAF application without any custom configuration options.

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
      caf_main(cfg);
    }

However, setting up config objects by hand is usually not necessary. CAFautomatically selects user-defined subclasses of ``actor_system_config`` if ``caf_main`` takes a second parameter by reference, as shown in the minimal example below.

::

    class my_config : public actor_system_config {
    public:
      void init() override {
        // ...
      }
    };

    void caf_main(actor_system& system, const my_config& cfg) {
      // ...
    }

    CAF_MAIN()

Users can perform additional initialization, add custom program options, etc. by overriding ``init``.

.. _system-config-module:

Loading Modules
---------------

The simplest way to load modules is to use the macro ``CAF_MAIN`` and pass a list of all requested modules, as sown below.

::

    void caf_main(actor_system& system) {
      // ...
    }
    CAF_MAIN(mod1, mod2, ...)

Alternatively, users can load modules in the ``init`` member function when implementing a custom config type.

::

    class my_config : public actor_system_config {
    public:
      void init() override {
        load<mod1>();
        load<mod2>();
        // ...
      }
    };

The third option is to simply call ``x.load<mod1>()`` on a config object *before* initializing an actor system with it.

.. _system-config-options:

Command Line Options and INI Configuration Files
------------------------------------------------

CAForganizes program options in categories and parses CLI arguments and INI files. CLI arguments override values in the INI file which override hard-coded defaults. Users can add any number of custom program options by implementing a subtype of ``actor_system_config`` and registering new options in ``init``. The example below adds three options to the “global” category.

::

    class config : public actor_system_config {
    public:
      uint16_t port = 0;
      std::string host = "localhost";
      bool server_mode = false;

      void init() override {
        opt_group{custom_options_, "global"}
        .add(port, "port,p", "set port")
        .add(host, "host,H", "set host (ignored in server mode)")
        .add(server_mode, "server-mode,s", "enable server mode");
      }
    };

The line ``opt_group{custom_options_, "global"}`` adds the “global” group to the list of custom options. The following calls to ``add`` then add individual options. The first argument to ``add`` is the associated variable. The second argument is the option name, optionally suffixed with a ``,`` and a single-character short name. The short name is only considered for CLI parsing and allows users to abbreviate commonly used option names. The third and final argument to ``add`` is a help text.

The custom ``config`` class allows end users to set the port for the application to 42 with either ``--port=42`` (long name) or ``-p 42`` (short name). The long option name is prefixed by the category when using a different category than “global”. For example, adding the port option to the category “foo” means end users have to type ``--foo.port=42`` when using the long name. Short names are unaffected by the category, but have to be unique.

Boolean options do not require arguments. The member variable ``server_mode`` is set to ``true`` if the command line contains either ``--server-mode`` or ``-s``.

CAFprefixes all of its default CLI options with ``caf#``, except for “help” (``--help``, ``-h``, or ``-?``). The default name for the INI file is ``caf-application.ini``. Users can change the file name and path by passing ``--caf#config-file=<path>`` on the command line.

INI files are organized in categories. No value is allowed outside of a category (no implicit “global” category). CAF reads ``true`` and ``false`` as boolean, numbers as (signed) integers or ``double``, ``"``-enclosed characters as strings, and ``'``-enclosed characters as atoms (see :ref:`atom`). The following example INI file lists all standard options in CAFand their default value. Note that some options such as ``scheduler.max-threads`` are usually detected at runtime and thus have no hard-coded default.

::

    ; This file shows all possible parameters with defaults.
    ; Values enclosed in <> are detected at runtime unless defined by the user.

    ; when using the default scheduler
    [scheduler]
    ; accepted alternative: 'sharing'
    policy='stealing'
    ; configures whether the scheduler generates profiling output
    enable-profiling=false
    ; can be overriden to force a fixed number of threads
    max-threads=<number of cores>
    ; configures the maximum number of messages actors can consume in one run
    max-throughput=<infinite>
    ; measurement resolution in milliseconds (only if profiling is enabled)
    profiling-ms-resolution=100
    ; output file for profiler data (only if profiling is enabled)
    profiling-output-file="/dev/null"

    ; when loading io::middleman
    [middleman]
    ; configures whether MMs try to span a full mesh
    enable-automatic-connections=false
    ; accepted alternative: 'asio' (only when compiling CAF with ASIO)
    network-backend='default'
    ; sets the maximum number of consecutive I/O reads per broker
    max-consecutive-reads=50
    ; heartbeat message interval in ms (0 disables heartbeating)
    heartbeat-interval=0

    ; when loading riac::probe
    [probe]
    ; denotes the hostname or IP address to reach the Nexus
    nexus-host=""
    ; denotes the port where the Nexus actor is published
    nexus-port=0

.. _adding-custom-message-types:

Adding Custom Message Types
---------------------------

CAFis designed with distributed systems in mind. Hence, all message types must be serializable and need a platform-neutral, unique name. Using a message type that is not serializable via a free function ``serialize`` causes a compiler error. Developers that use CAFfor concurrency only can suppress this error by whitelisting non-serializable message types using the macro ``CAF_ALLOW_UNSAFE_MESSAGE_TYPE``:

::

    #define CAF_ALLOW_UNSAFE_MESSAGE_TYPE(type_name)                               \
      namespace caf {                                                              \
      template <>                                                                  \
      struct allowed_unsafe_message_type<type_name> : std::true_type {};           \
      }

CAFserializes objects by calling ``serialize(proc, x, 0)``, where the data processor ``proc`` is either a serializer or a deserializer. The third parameter is a ``const unsigned int``, which is never evaluated by CAF. The parameter exists for source compatibility with ``Boost.Serialize``. As an introductory example, we use the following POD type ``foo``.

::

    struct foo {
      std::vector<int> a;
      int b;
    };

To make ``foo`` serializable, we implement a free function ``serialize``. Serializers provide ``operator<<``, while deserializers provide ``operator>>``. Both types also allow ``operator&`` to allow users to write a single function covering loading and storing, as shown below.

::

    template <class Processor>
    void serialize(Processor& proc, foo& x, const unsigned int) {
      proc & x.a;
      proc & x.b;
    }

Finally, we give ``foo`` a platform-neutral name and add it to the list of serializable types.

::

    class config : public actor_system_config {
    public:
      void init() override {
        add_message_type<foo>("foo");
      }
    };

    void caf_main(actor_system& system, const config&) {

If loading and storing cannot be implemented in a single function, users can query whether the processor is loading or storing as shown below.

::

    template <class T>
    typename std::enable_if<T::is_saving::value>::type
    serialize(T& out, const foo& x, const unsigned int) {
    template <class T>
    typename std::enable_if<T::is_loading::value>::type
    serialize(T& in, foo& x, const unsigned int) {

.. _adding-custom-error-types:

Adding Custom Error Types
-------------------------

Adding a custom error type to the system is a convenience feature to allow improve the string representation. Error types can be added by implementing a render function and passing it to ``add_error_category``, as shown in :ref:`custom-error`.
