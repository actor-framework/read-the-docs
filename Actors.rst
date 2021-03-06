.. _actor:

Actors
======



Actors in CAF are a lightweight abstraction for units of computations. They
are active objects in the sense that they own their state and do not allow
others to access it. The only way to modify the state of an actor is sending
messages to it.

CAF provides several actor implementations, each covering a particular use
case. The available implementations differ in three characteristics: (1)
dynamically or statically typed, (2) class-based or function-based, and (3)
using asynchronous event handlers or blocking receives. These three
characteristics can be combined freely, with one exception: statically typed
actors are always event-based. For example, an actor can have dynamically typed
messaging, implement a class, and use blocking receives. The common base class
for all user-defined actors is called ``local_actor``.

Dynamically typed actors are more familiar to developers coming from Erlang or
Akka. They (usually) enable faster prototyping but require extensive unit
testing. Statically typed actors require more source code but enable the
compiler to verify communication between actors. Since CAF supports both,
developers can freely mix both kinds of actors to get the best of both worlds.
A good rule of thumb is to make use of static type checking for actors that are
visible across multiple translation units.

Actors that utilize the blocking receive API always require an exclusive thread
of execution. Event-based actors, on the other hand, are usually scheduled
cooperatively and are very lightweight with a memory footprint of only few
hundred bytes. Developers can exclude---detach---event-based actors that
potentially starve others from the cooperative scheduling while spawning it. A
detached actor lives in its own thread of execution.

.. _actor-system:

Environment / Actor Systems
---------------------------



All actors live in an ``actor_system`` representing an actor
environment including scheduler scheduler_, registry registry_, and
optional components such as a middleman middleman_. A single process can
have multiple ``actor_system`` instances, but this is usually not
recommended (a use case for multiple systems is to strictly separate two or
more sets of actors by running them in different schedulers). For configuration
and fine-tuning options of actor systems see system-config_. A
distributed CAF application consists of two or more connected actor systems. We
also refer to interconnected ``actor_system`` instances as a
*distributed actor system*.

Common Actor Base Types
-----------------------



The following pseudo-UML depicts the class diagram for actors in CAF.
Irrelevant member functions and classes as well as mixins are omitted for
brevity. Selected individual classes are presented in more detail in the
following sections.

.. _actor-types:

.. image:: actor_types.png
   :alt: Actor Types in CAF



Class ``local_actor``
~~~~~~~~~~~~~~~~~~~~~



The class ``local_actor`` is the root type for all user-defined actors
in CAF. It defines all common operations. However, users of the library
usually do not inherit from this class directly. Proper base classes for
user-defined actors are ``event_based_actor`` or
``blocking_actor``. The following table also includes member function
inherited from ``monitorable_actor`` and ``abstract_actor``.



+-------------------------------------+--------------------------------------------------------+
| **Types**                           |                                                        |
+-------------------------------------+--------------------------------------------------------+
| ``mailbox_type``                    | A concurrent, many-writers-single-reader queue type.   |
+-------------------------------------+--------------------------------------------------------+
|                                     |                                                        |
+-------------------------------------+--------------------------------------------------------+
| **Constructors**                    |                                                        |
+-------------------------------------+--------------------------------------------------------+
| ``(actor_config&)``                 | Constructs the actor using a config.                   |
+-------------------------------------+--------------------------------------------------------+
|                                     |                                                        |
+-------------------------------------+--------------------------------------------------------+
| **Observers**                       |                                                        |
+-------------------------------------+--------------------------------------------------------+
| ``actor_addr address()``            | Returns the address of this actor.                     |
+-------------------------------------+--------------------------------------------------------+
| ``actor_system& system()``          | Returns ``context()->system()``.                       |
+-------------------------------------+--------------------------------------------------------+
| ``actor_system& home_system()``     | Returns the system that spawned this actor.            |
+-------------------------------------+--------------------------------------------------------+
| ``execution_unit* context()``       | Returns underlying thread or current scheduler worker. |
+-------------------------------------+--------------------------------------------------------+
|                                     |                                                        |
+-------------------------------------+--------------------------------------------------------+
| **Customization Points**            |                                                        |
+-------------------------------------+--------------------------------------------------------+
| ``on_exit()``                       | Can be overridden to perform cleanup code.             |
+-------------------------------------+--------------------------------------------------------+
| ``const char* name()``              | Returns a debug name for this actor type.              |
+-------------------------------------+--------------------------------------------------------+
|                                     |                                                        |
+-------------------------------------+--------------------------------------------------------+
| **Actor Management**                |                                                        |
+-------------------------------------+--------------------------------------------------------+
| ``link_to(other)``                  | Link to an actor link_.                                |
+-------------------------------------+--------------------------------------------------------+
| ``unlink_from(other)``              | Remove link to an actor link_.                         |
+-------------------------------------+--------------------------------------------------------+
| ``monitor(other)``                  | Unidirectionally monitors an actor monitor_.           |
+-------------------------------------+--------------------------------------------------------+
| ``demonitor(other)``                | Removes a monitor from ``whom``.                       |
+-------------------------------------+--------------------------------------------------------+
| ``spawn(F fun, xs...)``             | Spawns a new actor from ``fun``.                       |
+-------------------------------------+--------------------------------------------------------+
| ``spawn<T>(xs...)``                 | Spawns a new actor of type ``T``.                      |
+-------------------------------------+--------------------------------------------------------+
|                                     |                                                        |
+-------------------------------------+--------------------------------------------------------+
| **Message Processing**              |                                                        |
+-------------------------------------+--------------------------------------------------------+
| ``T make_response_promise<Ts...>()``| Allows an actor to delay its response message.         |
+-------------------------------------+--------------------------------------------------------+
| ``T response(xs...)``               | Convenience function for creating fulfilled promises.  |
+-------------------------------------+--------------------------------------------------------+


Class ``scheduled_actor``
~~~~~~~~~~~~~~~~~~~~~~~~~



All scheduled actors inherit from ``scheduled_actor``. This includes
statically and dynamically typed event-based actors as well as brokers
broker_.



+-------------------------------+--------------------------------------------------------------------------+
| **Types**                     |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| ``pointer``                   | ``scheduled_actor*``                                                     |
+-------------------------------+--------------------------------------------------------------------------+
| ``exception_handler``         | ``function<error (pointer, std::exception_ptr&)>``                       |
+-------------------------------+--------------------------------------------------------------------------+
| ``default_handler``           | ``function<result<message> (pointer, message_view&)>``                   |
+-------------------------------+--------------------------------------------------------------------------+
| ``error_handler``             | ``function<void (pointer, error&)>``                                     |
+-------------------------------+--------------------------------------------------------------------------+
| ``down_handler``              | ``function<void (pointer, down_msg&)>``                                  |
+-------------------------------+--------------------------------------------------------------------------+
| ``exit_handler``              | ``function<void (pointer, exit_msg&)>``                                  |
+-------------------------------+--------------------------------------------------------------------------+
|                               |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| **Constructors**              |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| ``(actor_config&)``           | Constructs the actor using a config.                                     |
+-------------------------------+--------------------------------------------------------------------------+
|                               |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| **Termination**               |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| ``quit()``                    | Stops this actor with normal exit reason.                                |
+-------------------------------+--------------------------------------------------------------------------+
| ``quit(error x)``             | Stops this actor with error ``x``.                                       |
+-------------------------------+--------------------------------------------------------------------------+
|                               |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| **Special-purpose Handlers**  |                                                                          |
+-------------------------------+--------------------------------------------------------------------------+
| ``set_exception_handler(F f)``| Installs ``f`` for converting exceptions to errors error_.               |
+-------------------------------+--------------------------------------------------------------------------+
| ``set_down_handler(F f)``     | Installs ``f`` to handle down messages down-message_.                    |
+-------------------------------+--------------------------------------------------------------------------+
| ``set_exit_handler(F f)``     | Installs ``f`` to handle exit messages exit-message_.                    |
+-------------------------------+--------------------------------------------------------------------------+
| ``set_error_handler(F f)``    | Installs ``f`` to handle error messages (see error-message_ and error_). |
+-------------------------------+--------------------------------------------------------------------------+
| ``set_default_handler(F f)``  | Installs ``f`` as fallback message handler default-handler_.             |
+-------------------------------+--------------------------------------------------------------------------+


Class ``blocking_actor``
~~~~~~~~~~~~~~~~~~~~~~~~



A blocking actor always lives in its own thread of execution. They are not as
lightweight as event-based actors and thus do not scale up to large numbers.
The primary use case for blocking actors is to use a ``scoped_actor``
for ad-hoc communication to selected actors. Unlike scheduled actors, CAF does
**not** dispatch system messages to special-purpose handlers. A blocking
actors receives *all* messages regularly through its mailbox. A blocking
actor is considered *done* only after it returned from ``act`` (or
from the implementation in function-based actors). A ``scoped_actor``
sends its exit messages as part of its destruction.



+----------------------------------+---------------------------------------------------+
| **Constructors**                 |                                                   |
+----------------------------------+---------------------------------------------------+
| ``(actor_config&)``              | Constructs the actor using a config.              |
+----------------------------------+---------------------------------------------------+
|                                  |                                                   |
+----------------------------------+---------------------------------------------------+
| **Customization Points**         |                                                   |
+----------------------------------+---------------------------------------------------+
| ``void act()``                   | Implements the behavior of the actor.             |
+----------------------------------+---------------------------------------------------+
|                                  |                                                   |
+----------------------------------+---------------------------------------------------+
| **Termination**                  |                                                   |
+----------------------------------+---------------------------------------------------+
| ``const error& fail_state()``    | Returns the current exit reason.                  |
+----------------------------------+---------------------------------------------------+
| ``fail_state(error x)``          | Sets the current exit reason.                     |
+----------------------------------+---------------------------------------------------+
|                                  |                                                   |
+----------------------------------+---------------------------------------------------+
| **Actor Management**             |                                                   |
+----------------------------------+---------------------------------------------------+
| ``wait_for(Ts... xs)``           | Blocks until all actors ``xs...`` are done.       |
+----------------------------------+---------------------------------------------------+
| ``await_all_other_actors_done()``| Blocks until all other actors are done.           |
+----------------------------------+---------------------------------------------------+
|                                  |                                                   |
+----------------------------------+---------------------------------------------------+
| **Message Handling**             |                                                   |
+----------------------------------+---------------------------------------------------+
| ``receive(Ts... xs)``            | Receives a message using the callbacks ``xs...``. |
+----------------------------------+---------------------------------------------------+
| ``receive_for(T& begin, T end)`` | See receive-loop_.                                |
+----------------------------------+---------------------------------------------------+
| ``receive_while(F stmt)``        | See receive-loop_.                                |
+----------------------------------+---------------------------------------------------+
| ``do_receive(Ts... xs)``         | See receive-loop_.                                |
+----------------------------------+---------------------------------------------------+


.. _interface:

Messaging Interfaces
--------------------



Statically typed actors require abstract messaging interfaces to allow the
compiler to type-check actor communication. Interfaces in CAF are defined using
the variadic template ``typed_actor<...>``, which defines the proper
actor handle at the same time. Each template parameter defines one
``input/output`` pair via
``replies_to<X1,...,Xn>::with<Y1,...,Yn>``. For inputs that do not
generate outputs, ``reacts_to<X1,...,Xn>`` can be used as shortcut for
``replies_to<X1,...,Xn>::with<void>``. In the same way functions cannot
be overloaded only by their return type, interfaces cannot accept one input
twice (possibly mapping it to different outputs). The example below defines a
messaging interface for a simple calculator.


.. code-block:: c++

   using calculator_actor = typed_actor<replies_to<add_atom, int, int>::with<int>,




It is not required to create a type alias such as ``calculator_actor``,
but it makes dealing with statically typed actors much easier. Also, a central
alias definition eases refactoring later on.

Interfaces have set semantics. This means the following two type aliases
``i1`` and ``i2`` are equal:


.. code-block:: C++

   using i1 = typed_actor<replies_to<A>::with<B>, replies_to<C>::with<D>>;
   using i2 = typed_actor<replies_to<C>::with<D>, replies_to<A>::with<B>>;



Further, actor handles of type ``A`` are assignable to handles of type
``B`` as long as ``B`` is a subset of ``A``.

For convenience, the class ``typed_actor<...>`` defines the member
types shown below to grant access to derived types.



+------------------------+---------------------------------------------------------------+
| **Types**              |                                                               |
+------------------------+---------------------------------------------------------------+
| ``behavior_type``      | A statically typed set of message handlers.                   |
+------------------------+---------------------------------------------------------------+
| ``base``               | Base type for actors, i.e., ``typed_event_based_actor<...>``. |
+------------------------+---------------------------------------------------------------+
| ``pointer``            | A pointer of type ``base*``.                                  |
+------------------------+---------------------------------------------------------------+
| ``stateful_base<T>``   | See stateful-actor_.                                          |
+------------------------+---------------------------------------------------------------+
| ``stateful_pointer<T>``| A pointer of type ``stateful_base<T>*``.                      |
+------------------------+---------------------------------------------------------------+
| ``extend<Ts...>``      | Extend this typed actor with ``Ts...``.                       |
+------------------------+---------------------------------------------------------------+
| ``extend_with<Other>`` | Extend this typed actor with all cases from ``Other``.        |
+------------------------+---------------------------------------------------------------+


.. _spawn:

Spawning Actors
---------------



Both statically and dynamically typed actors are spawned from an
``actor_system`` using the member function ``spawn``. The
function either takes a function as first argument or a class as first template
parameter. For example, the following functions and classes represent actors.


.. code-block:: c++

   behavior calculator_fun(event_based_actor* self);
   void blocking_calculator_fun(blocking_actor* self);
   calculator_actor::behavior_type typed_calculator_fun();
   class calculator;
   class blocking_calculator;




Spawning an actor for each implementation is illustrated below.


.. code-block:: c++

     auto a1 = system.spawn(blocking_calculator_fun);
     auto a2 = system.spawn(calculator_fun);
     auto a3 = system.spawn(typed_calculator_fun);
     auto a4 = system.spawn<blocking_calculator>();
     auto a5 = system.spawn<calculator>();




Additional arguments to ``spawn`` are passed to the constructor of a
class or used as additional function arguments, respectively. In the example
above, none of the three functions takes any argument other than the implicit
but optional ``self`` pointer.

.. _function-based:

Function-based Actors
---------------------



When using a function or function object to implement an actor, the first
argument *can* be used to capture a pointer to the actor itself. The type
of this pointer is usually ``event_based_actor*`` or
``blocking_actor*``. The proper pointer type for any
``typed_actor`` handle ``T`` can be obtained via
``T::pointer`` interface_.

Blocking actors simply implement their behavior in the function body. The actor
is done once it returns from that function.

Event-based actors can either return a ``behavior``
message-handler_ that is used to initialize the actor or explicitly set the
initial behavior by calling ``self->become(...)``. Due to the
asynchronous, event-based nature of this kind of actor, the function usually
returns immediately after setting a behavior (message handler) for the
*next* incoming message. Hence, variables on the stack will be out of scope
once a message arrives. Managing state in function-based actors can be done
either via rebinding state with ``become``, using heap-located data
referenced via ``std::shared_ptr`` or by using the *stateful
actor* abstraction stateful-actor_.

The following three functions implement the prototypes shown in spawn_
and illustrate one blocking actor and two event-based actors (statically and
dynamically typed).


.. code-block:: c++

   // function-based, dynamically typed, event-based API
   behavior calculator_fun(event_based_actor*) {
     return {
       [](add_atom, int a, int b) { return a + b; },
       [](sub_atom, int a, int b) { return a - b; },
     };
   }
   
   // function-based, dynamically typed, blocking API
   void blocking_calculator_fun(blocking_actor* self) {
     bool running = true;
     self->receive_while(running)( //
       [](add_atom, int a, int b) { return a + b; },
       [](sub_atom, int a, int b) { return a - b; },
       [&](exit_msg& em) {
         if (em.reason) {
           self->fail_state(std::move(em.reason));
           running = false;
         }
       });
   }
   
   // function-based, statically typed, event-based API
   calculator_actor::behavior_type typed_calculator_fun() {
     return {
       [](add_atom, int a, int b) { return a + b; },
       [](sub_atom, int a, int b) { return a - b; },
     };




.. _class-based:

Class-based Actors
------------------



Implementing an actor using a class requires the following:


* Provide a constructor taking a reference of type  ``actor_config&`` as first argument, which is forwarded to the base  class. The config is passed implicitly to the constructor when calling  ``spawn``, which also forwards any number of additional arguments  to the constructor.
* Override ``make_behavior`` for event-based actors and  ``act`` for blocking actors.



Implementing actors with classes works for all kinds of actors and allows
simple management of state via member variables. However, composing states via
inheritance can get quite tedious. For dynamically typed actors, composing
states is particularly hard, because the compiler cannot provide much help. For
statically typed actors, CAF also provides an API for composable
behaviors composable-behavior_ that works well with inheritance. The
following three examples implement the forward declarations shown in
spawn_.


.. code-block:: c++

   // class-based, dynamically typed, event-based API
   class calculator : public event_based_actor {
   public:
     calculator(actor_config& cfg) : event_based_actor(cfg) {
       // nop
     }
   
     behavior make_behavior() override {
       return calculator_fun(this);
     }
   };
   
   // class-based, dynamically typed, blocking API
   class blocking_calculator : public blocking_actor {
   public:
     blocking_calculator(actor_config& cfg) : blocking_actor(cfg) {
       // nop
     }
   
     void act() override {
       blocking_calculator_fun(this);
     }
   };
   
   // class-based, statically typed, event-based API
   class typed_calculator : public calculator_actor::base {
   public:
     typed_calculator(actor_config& cfg) : calculator_actor::base(cfg) {
       // nop
     }
   
     behavior_type make_behavior() override {
       return typed_calculator_fun();
     }




.. _stateful-actor:

Stateful Actors
---------------



The stateful actor API makes it easy to maintain state in function-based
actors. It is also safer than putting state in member variables, because the
state ceases to exist after an actor is done and is not delayed until the
destructor runs. For example, if two actors hold a reference to each other via
member variables, they produce a cycle and neither will get destroyed. Using
stateful actors instead breaks the cycle, because references are destroyed when
an actor calls ``self->quit()`` (or is killed externally). The
following example illustrates how to implement stateful actors with static
typing as well as with dynamic typing.


.. code-block:: c++

   using cell = typed_actor<reacts_to<put_atom, int>,
                            replies_to<get_atom>::with<int>>;
   
   struct cell_state {
     int value = 0;
   };
   
   cell::behavior_type type_checked_cell(cell::stateful_pointer<cell_state> self) {
     return {
       [=](put_atom, int val) {
         self->state.value = val;
       },
       [=](get_atom) {
         return self->state.value;
       }
     };
   }
   
   behavior unchecked_cell(stateful_actor<cell_state>* self) {
     return {
       [=](put_atom, int val) {
         self->state.value = val;
       },
       [=](get_atom) {
         return self->state.value;
       }




Stateful actors are spawned in the same way as any other function-based actor
function-based_.


.. code-block:: c++

     auto cell1 = system.spawn(type_checked_cell);




.. _composable-behavior:

Actors from Composable Behaviors  :sup:`experimental`
-----------------------------------------------------



When building larger systems, it is often useful to implement the behavior of
an actor in terms of other, existing behaviors. The composable behaviors in
CAF allow developers to generate a behavior class from a messaging
interface interface_.

The base type for composable behaviors is ``composable_behavior<T>``,
where ``T`` is a ``typed_actor<...>``. CAF maps each
``replies_to<A,B,C>::with<D,E,F>`` in ``T`` to a pure virtual
member function with signature:


.. code-block:: C++

     result<D, E, F> operator()(param<A>, param<B>, param<C>);.



Note that ``operator()`` will take integral types as well as atom
constants simply by value. A ``result<T>`` accepts either a value of
type ``T``, a ``skip_t`` default-handler_, an
``error`` error_, a ``delegated<T>`` delegate_, or a
``response_promise<T>`` promise_. A ``result<void>`` is
constructed by returning ``unit``.

A behavior that combines the behaviors ``X``, ``Y``, and
``Z`` must inherit from ``composed_behavior<X,Y,Z>`` instead of
inheriting from the three classes directly. The class
``composed_behavior`` ensures that the behaviors are concatenated
correctly. In case one message handler is defined in multiple base types, the
*first* type in declaration order wins. For example, if ``X`` and
``Y`` both implement the interface
``replies_to<int,int>::with<int>``, only the handler implemented in
``X`` is active.

Any composable (or composed) behavior with no pure virtual member functions can
be spawned directly through an actor system by calling
``system.spawn<...>()``, as shown below.


.. code-block:: c++

   using adder = typed_actor<replies_to<add_atom, int, int>::with<int>>;
   
   using multiplier = typed_actor<replies_to<mul_atom, int, int>::with<int>>;
   
   class adder_bhvr : public composable_behavior<adder> {
   public:
     result<int> operator()(add_atom, int x, int y) override {
       return x + y;
     }
   };
   
   class multiplier_bhvr : public composable_behavior<multiplier> {
   public:
     result<int> operator()(mul_atom, int x, int y) override {
       return x * y;
     }
   };
   
   // calculator_bhvr can be inherited from or composed further
   using calculator_bhvr = composed_behavior<adder_bhvr, multiplier_bhvr>;
   
   void caf_main(actor_system& system) {
     auto f = make_function_view(system.spawn<calculator_bhvr>());
     cout << "10 + 20 = " << f(add_atom_v, 10, 20) << endl;
     cout << "7 * 9 = " << f(mul_atom_v, 7, 9) << endl;




The second example illustrates how to use non-primitive values that are wrapped
in a ``param<T>`` when working with composable behaviors. The purpose
of ``param<T>`` is to provide a single interface for both constant and
non-constant access. Constant access is modeled with the implicit conversion
operator to a const reference, the member function ``get()``, and
``operator->``.

When acquiring mutable access to the represented value, CAF copies the value
before allowing mutable access to it if more than one reference to the value
exists. This copy-on-write optimization avoids race conditions by design, while
minimizing copy operations copy-on-write_. A mutable reference is returned
from the member functions ``get_mutable()`` and ``move()``. The
latter is a convenience function for ``std::move(x.get_mutable())``.
The following example illustrates how to use ``param<std::string>``
when implementing a simple dictionary.


.. code-block:: c++

   using dict = typed_actor<reacts_to<put_atom, string, string>,
                            replies_to<get_atom, string>::with<string>>;
   
   class dict_behavior : public composable_behavior<dict> {
   public:
     result<string> operator()(get_atom, param<string> key) override {
       auto i = values_.find(key);
       if (i == values_.end())
         return "";
       return i->second;
     }
   
     result<void> operator()(put_atom, param<string> key,
                             param<string> value) override {
       if (values_.count(key) != 0)
         return unit;
       values_.emplace(key.move(), value.move());
       return unit;
     }
   
   protected:
     std::unordered_map<string, string> values_;




.. _attach:

Attaching Cleanup Code to Actors
--------------------------------



Users can attach cleanup code to actors. This code is executed immediately if
the actor has already exited. Otherwise, the actor will execute it as part of
its termination. The following example attaches a function object to actors for
printing a custom string on exit.


.. code-block:: c++

   // Utility function to print an exit message with custom name.
   void print_on_exit(const actor& hdl, const std::string& name) {
     hdl->attach_functor([=](const error& reason) {
       cout << name << " exited: " << to_string(reason) << endl;
     });




It is possible to attach code to remote actors. However, the cleanup code will
run on the local machine.

.. _blocking-actor:

Blocking Actors
---------------



Blocking actors always run in a separate thread and are not scheduled by CAF.
Unlike event-based actors, blocking actors have explicit, blocking
*receive* functions. Further, blocking actors do not handle system
messages automatically via special-purpose callbacks special-handler_.
This gives users full control over the behavior of blocking actors. However,
blocking actors still should follow conventions of the actor system. For
example, actors should unconditionally terminate after receiving an
``exit_msg`` with reason ``exit_reason::kill``.

Receiving Messages
~~~~~~~~~~~~~~~~~~



The function ``receive`` sequentially iterates over all elements in the
mailbox beginning with the first. It takes a message handler that is applied to
the elements in the mailbox until an element was matched by the handler. An
actor calling ``receive`` is blocked until it successfully dequeued a
message from its mailbox or an optional timeout occurs. Messages that are not
matched by the behavior are automatically skipped and remain in the mailbox.


.. code-block:: C++

   self->receive (
     [](int x) { /* ... */ }
   );



.. _catch-all:

Catch-all Receive Statements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~



Blocking actors can use inline catch-all callbacks instead of setting a default
handler default-handler_. A catch-all case must be the last callback
before the optional timeout, as shown in the example below.


.. code-block:: C++

   self->receive(
     [&](float x) {
       // ...
     },
     [&](const down_msg& x) {
       // ...
     },
     [&](const exit_msg& x) {
       // ...
     },
     others >> [](message_view& x) -> result<message> {
       // report unexpected message back to client
       return sec::unexpected_message;
     }
   );



.. _receive-loop:

Receive Loops
~~~~~~~~~~~~~



Message handler passed to ``receive`` are temporary object at runtime.
Hence, calling ``receive`` inside a loop creates an unnecessary amount
of short-lived objects. CAF provides predefined receive loops to allow for
more efficient code.


.. code-block:: C++

   // BAD
   std::vector<int> results;
   for (size_t i = 0; i < 10; ++i)
     receive (
       [&](int value) {
         results.push_back(value);
       }
     );
   
   // GOOD
   std::vector<int> results;
   size_t i = 0;
   receive_for(i, 10) (
     [&](int value) {
       results.push_back(value);
     }
   );




.. code-block:: C++

   // BAD
   size_t received = 0;
   while (received < 10) {
     receive (
       [&](int) {
         ++received;
       }
     );
   } ;
   
   // GOOD
   size_t received = 0;
   receive_while([&] { return received < 10; }) (
     [&](int) {
       ++received;
     }
   );



.. code-block:: C++

   // BAD
   size_t received = 0;
   do {
     receive (
       [&](int) {
         ++received;
       }
     );
   } while (received < 10);
   
   // GOOD
   size_t received = 0;
   do_receive (
     [&](int) {
       ++received;
     }
   ).until([&] { return received >= 10; });



The examples above illustrate the correct usage of the three loops
``receive_for``, ``receive_while`` and
``do_receive(...).until``. It is possible to nest receives and receive
loops.


.. code-block:: C++

   bool running = true;
   self->receive_while([&] { return running; }) (
     [&](int value1) {
       self->receive (
         [&](float value2) {
           aout(self) << value1 << " => " << value2 << endl;
         }
       );
     },
     // ...
   );



.. _scoped-actors:

Scoped Actors
~~~~~~~~~~~~~



The class ``scoped_actor`` offers a simple way of communicating with
CAF actors from non-actor contexts. It overloads ``operator->`` to
return a ``blocking_actor*``. Hence, it behaves like the implicit
``self`` pointer in functor-based actors, only that it ceases to exist
at scope end.


.. code-block:: C++

   void test(actor_system& system) {
     scoped_actor self{system};
     // spawn some actor
     auto aut = self->spawn(my_actor_impl);
     self->send(aut, "hi there");
     // self will be destroyed automatically here; any
     // actor monitoring it will receive down messages etc.
   }


