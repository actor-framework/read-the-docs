.. _error:

Errors
======



Errors in CAF have a code and a category, similar to
``std::error_code`` and ``std::error_condition``. Unlike its
counterparts from the C++ standard library, ``error`` is
plattform-neutral and serializable. Instead of using category singletons, CAF
stores categories as atoms atom_. Errors can also include a message to
provide additional context information.

Class Interface
---------------



+-----------------------------------------+--------------------------------------------------------------------+
| **Constructors**                        |                                                                    |
+-----------------------------------------+--------------------------------------------------------------------+
| ``(Enum x)``                            | Construct error by calling ``make_error(x)``                       |
+-----------------------------------------+--------------------------------------------------------------------+
| ``(uint8_t x, atom_value y)``           | Construct error with code ``x`` and category ``y``                 |
+-----------------------------------------+--------------------------------------------------------------------+
| ``(uint8_t x, atom_value y, message z)``| Construct error with code ``x``, category ``y``, and context ``z`` |
+-----------------------------------------+--------------------------------------------------------------------+
|                                         |                                                                    |
+-----------------------------------------+--------------------------------------------------------------------+
| **Observers**                           |                                                                    |
+-----------------------------------------+--------------------------------------------------------------------+
| ``uint8_t code()``                      | Returns the error code                                             |
+-----------------------------------------+--------------------------------------------------------------------+
| ``atom_value category()``               | Returns the error category                                         |
+-----------------------------------------+--------------------------------------------------------------------+
| ``message context()``                   | Returns additional context information                             |
+-----------------------------------------+--------------------------------------------------------------------+
| ``explicit operator bool()``            | Returns ``code() != 0``                                            |
+-----------------------------------------+--------------------------------------------------------------------+


.. _custom-error:

Add Custom Error Categories
---------------------------



Adding custom error categories requires three steps: (1) declare an enum class
of type ``uint8_t`` with the first value starting at 1, (2) implement a
free function ``make_error`` that converts the enum to an
``error`` object, (3) add the custom category to the actor system with
a render function. The last step is optional to allow users to retrieve a
better string representation from ``system.render(x)`` than
``to_string(x)`` can offer. Note that any error code with value 0 is
interpreted as *not-an-error*. The following example adds a custom error
category by performing the first two steps.


.. code-block:: c++

   enum class math_error : uint8_t {
     division_by_zero = 1
   };
   
   error make_error(math_error x) {
     return {static_cast<uint8_t>(x), atom("math")};
   }
   
   std::string to_string(math_error x) {
     switch (x) {
       case math_error::division_by_zero:
         return "division_by_zero";
       default:
         return "-unknown-error-";
     }




The implementation of ``to_string(error)`` is unable to call string
conversions for custom error categories. Hence,
``to_string(make_error(math_error::division_by_zero))`` returns
``"error(1, math)"``.

The following code adds a rendering function to the actor system to provide a
more satisfactory string conversion.


.. code-block:: c++

   class config : public actor_system_config {
   public:
     config() {
       auto renderer = [](uint8_t x, atom_value, const message&) {
         return "math_error" + deep_to_string_as_tuple(static_cast<math_error>(x));
       };
       add_error_category(atom("math"), renderer);
     }




With the custom rendering function,
``system.render(make_error(math_error::division_by_zero))`` returns
``"math_error(division_by_zero)"``.

.. _sec:

System Error Codes
------------------



System Error Codes (SECs) use the error category ``"system"``. They
represent errors in the actor system or one of its modules and are defined as
follows.


.. code-block:: c++

   /// SEC stands for "System Error Code". This enum contains error codes for
   /// ::actor_system and its modules.
   enum class sec : uint8_t {
     /// No error.
     none = 0,
     /// Indicates that an actor dropped an unexpected message.
     unexpected_message = 1,
     /// Indicates that a response message did not match the provided handler.
     unexpected_response,
     /// Indicates that the receiver of a request is no longer alive.
     request_receiver_down,
     /// Indicates that a request message timed out.
     request_timeout,
     /// Indicates that requested group module does not exist.
     no_such_group_module = 5,
     /// Unpublishing or connecting failed: no actor bound to given port.
     no_actor_published_at_port,
     /// Connecting failed because a remote actor had an unexpected interface.
     unexpected_actor_messaging_interface,
     /// Migration failed because the state of an actor is not serializable.
     state_not_serializable,
     /// An actor received an unsupported key for `('sys', 'get', key)` messages.
     unsupported_sys_key,
     /// An actor received an unsupported system message.
     unsupported_sys_message = 10,
     /// A remote node disconnected during CAF handshake.
     disconnect_during_handshake,
     /// Tried to forward a message via BASP to an invalid actor handle.
     cannot_forward_to_invalid_actor,
     /// Tried to forward a message via BASP to an unknown node ID.
     no_route_to_receiving_node,
     /// Middleman could not assign a connection handle to a broker.
     failed_to_assign_scribe_from_handle,
     /// Middleman could not assign an acceptor handle to a broker.
     failed_to_assign_doorman_from_handle = 15,
     /// User requested to close port 0 or to close a port not managed by CAF.
     cannot_close_invalid_port,
     /// Middleman could not connect to a remote node.
     cannot_connect_to_node,
     /// Middleman could not open requested port.
     cannot_open_port,
     /// A C system call in the middleman failed.
     network_syscall_failed,
     /// A function received one or more invalid arguments.
     invalid_argument = 20,
     /// A network socket reported an invalid network protocol family.
     invalid_protocol_family,
     /// Middleman could not publish an actor because it was invalid.
     cannot_publish_invalid_actor,
     /// A remote spawn failed because the provided types did not match.
     cannot_spawn_actor_from_arguments,
     /// Serialization failed because there was not enough data to read.
     end_of_stream,
     /// Serialization failed because no CAF context is available.
     no_context = 25,
     /// Serialization failed because CAF misses run-time type information.
     unknown_type,
     /// Serialization of actors failed because no proxy registry is available.
     no_proxy_registry,
     /// An exception was thrown during message handling.
     runtime_error,
     /// Linking to a remote actor failed because actor no longer exists.
     remote_linking_failed,
     /// Adding an upstream to a stream failed.
     cannot_add_upstream = 30,
     /// Adding an upstream to a stream failed because it already exists.
     upstream_already_exists,
     /// Unable to process upstream messages because upstream is invalid.
     invalid_upstream,
     /// Adding a downstream to a stream failed.
     cannot_add_downstream,
     /// Adding a downstream to a stream failed because it already exists.
     downstream_already_exists,
     /// Unable to process downstream messages because downstream is invalid.
     invalid_downstream = 35,
     /// Cannot start streaming without next stage.
     no_downstream_stages_defined,
     /// Actor failed to initialize state after receiving a stream handshake.
     stream_init_failed,
     /// Unable to process a stream since due to missing state.
     invalid_stream_state,
     /// Stream aborted due to unexpected error.
     unhandled_stream_error,
     /// A function view was called without assigning an actor first.
     bad_function_call = 40,




.. _exit-reason:

Default Exit Reasons
--------------------



CAF uses the error category ``"exit"`` for default exit reasons. These
errors are usually fail states set by the actor system itself. The two
exceptions are ``exit_reason::user_shutdown`` and
``exit_reason::kill``. The former is used in CAF to signalize orderly,
user-requested shutdown and can be used by programmers in the same way. The
latter terminates an actor unconditionally when used in ``send_exit``,
even if the default handler for exit messages exit-message_ is overridden.


.. code-block:: c++

   
   namespace caf {
   
   /// This error category represents fail conditions for actors.
   enum class exit_reason : uint8_t {
     /// Indicates that an actor finished execution without error.
     normal = 0,
     /// Indicates that an actor died because of an unhandled exception.
     unhandled_exception,
     /// Indicates that the exit reason for this actor is unknown, i.e.,
     /// the actor has been terminated and no longer exists.
     unknown,
     /// Indicates that an actor pool unexpectedly ran out of workers.
     out_of_workers,
     /// Indicates that an actor was forced to shutdown by a user-generated event.
     user_shutdown,
     /// Indicates that an actor was killed unconditionally.
     kill,
     /// Indicates that an actor finishied execution because a connection
     /// to a remote link was closed unexpectedly.



