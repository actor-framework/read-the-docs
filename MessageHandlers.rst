.. _message-handler:

Message Handlers
================



Actors can store a set of callbacks---usually implemented as lambda
expressions---using either ``behavior`` or ``message_handler``.
The former stores an optional timeout, while the latter is composable.

Definition and Composition
--------------------------



As the name implies, a ``behavior`` defines the response of an actor to
messages it receives. The optional timeout allows an actor to dynamically
change its behavior when not receiving message after a certain amount of time.


.. code-block:: C++

   message_handler x1{
     [](int i) { /*...*/ },
     [](double db) { /*...*/ },
     [](int a, int b, int c) { /*...*/ }
   };



In our first example, ``x1`` models a behavior accepting messages that
consist of either exactly one ``int``, or one ``double``, or
three ``int`` values. Any other message is not matched and gets
forwarded to the default handler default-handler_.


.. code-block:: C++

   message_handler x2{
     [](double db) { /*...*/ },
     [](double db) { /* - unreachable - */ }
   };



Our second example illustrates an important characteristic of the matching
mechanism. Each message is matched against the callbacks in the order they are
defined. The algorithm stops at the first match. Hence, the second callback in
``x2`` is unreachable.


.. code-block:: C++

   message_handler x3 = x1.or_else(x2);
   message_handler x4 = x2.or_else(x1);



Message handlers can be combined using ``or_else``. This composition is
not commutative, as our third examples illustrates. The resulting message
handler will first try to handle a message using the left-hand operand and will
fall back to the right-hand operand if the former did not match. Thus,
``x3`` behaves exactly like ``x1``. This is because the second
callback in ``x1`` will consume any message with a single
``double`` and both callbacks in ``x2`` are thus unreachable.
The handler ``x4`` will consume messages with a single
``double`` using the first callback in ``x2``, essentially
overriding the second callback in ``x1``.

.. _atom:

Atoms
-----



Defining message handlers in terms of callbacks is convenient, but requires a
simple way to annotate messages with meta data. Imagine an actor that provides
a mathematical service for integers. It receives two integers, performs a
user-defined operation and returns the result. Without additional context, the
actor cannot decide whether it should multiply or add the integers. Thus, the
operation must be encoded into the message. The Erlang programming language
introduced an approach to use non-numerical constants, so-called
*atoms*, which have an unambiguous, special-purpose type and do not have
the runtime overhead of string constants.

Atoms in CAF are mapped to integer values at compile time. This mapping is
guaranteed to be collision-free and invertible, but limits atom literals to ten
characters and prohibits special characters. Legal characters are
``_0-9A-Za-z`` and the whitespace character. Atoms are created using
the ``constexpr`` function ``atom``, as the following example
illustrates.


.. code-block:: C++

   atom_value a1 = atom("add");
   atom_value a2 = atom("multiply");



**Warning**: The compiler cannot enforce the restrictions at compile time,
except for a length check. The assertion ``atom("!?") != atom("?!")``
is not true, because each invalid character translates to a whitespace
character.

While the ``atom_value`` is computed at compile time, it is not
uniquely typed and thus cannot be used in the signature of a callback. To
accomplish this, CAF offers compile-time *atom constants*.


.. code-block:: C++

   using add_atom = atom_constant<atom("add")>;
   using multiply_atom = atom_constant<atom("multiply")>;



Using these constants, we can now define message passing interfaces in a
convenient way:


.. code-block:: C++

   behavior do_math{
     [](add_atom, int a, int b) {
       return a + b;
     },
     [](multiply_atom, int a, int b) {
       return a * b;
     }
   };
   
   // caller side: send(math_actor, add_atom::value, 1, 2)



Atom constants define a static member ``value``. Please note that this
static ``value`` member does *not* have the type
``atom_value``, unlike ``std::integral_constant`` for example.
