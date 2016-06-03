.. _message:

Type-Erased Tuples and Messages
===============================

Messages in CAFare type-erased, copy-on-write tuples. The actual message type itself is usually hidden, as actors use pattern matching to decompose messages automatically. However, the classes ``message`` and ``message_builder`` allow more advanced usage scenarios than only sending data from one actor to another.

.. _rtti-and-type-numbers:

RTTI and Type Numbers
---------------------

All builtin types in CAFhave a non-zero 16-bit *type number*. All user-defined types are mapped to 0. When querying the run-time type information (RTTI) for individual message or tuple elements, CAFreturns a ``std::pair<uint16_t, const std::type_info*>``. The first value is the 16-bit type number. If the type number is non-zero, the second value is a pointer to the C++ type info, otherwise the second value is null.

.. _class-type_erased_tuple:

Class ``type_erased_tuple``
---------------------------

+------------------------------------------+----------------------------------------------------------+
| **Types**                                |                                                          |
+==========================================+==========================================================+
| ``rtti_pair``                            | ``std::pair<uint16_t, const std::type_info*>``           |
+------------------------------------------+----------------------------------------------------------+
|                                          |                                                          |
+------------------------------------------+----------------------------------------------------------+
| **Observers**                            |                                                          |
+------------------------------------------+----------------------------------------------------------+
| ``bool empty()``                         | Returns whether this message is empty.                   |
+------------------------------------------+----------------------------------------------------------+
| ``size_t size()``                        | Returns the size of this message.                        |
+------------------------------------------+----------------------------------------------------------+
| ``rtti_pair type(size_t pos)``           | Returns run-time type information for the nth element.   |
+------------------------------------------+----------------------------------------------------------+
| ``void save(serializer& x)``             | Writes the tuple to ``x``.                               |
+------------------------------------------+----------------------------------------------------------+
| ``void save(size_t n, serializer& x)``   | Writes the nth element to ``x``.                         |
+------------------------------------------+----------------------------------------------------------+
| ``const void* get(size_t n)``            | Returns a const pointer to the nth element.              |
+------------------------------------------+----------------------------------------------------------+
| ``std::string stringify()``              | Returns a string representation of the tuple.            |
+------------------------------------------+----------------------------------------------------------+
| ``std::string stringify(size_t n)``      | Returns a string representation of the nth element.      |
+------------------------------------------+----------------------------------------------------------+
| ``bool matches(size_t n, rtti_pair)``    | Checks whether the nth element has given type.           |
+------------------------------------------+----------------------------------------------------------+
|                                          |                                                          |
+------------------------------------------+----------------------------------------------------------+
| **Modifiers**                            |                                                          |
+------------------------------------------+----------------------------------------------------------+
| ``void* get_mutable(size_t n)``          | Returns a mutable pointer to the nth element.            |
+------------------------------------------+----------------------------------------------------------+
| ``void load(deserializer& x)``           | Reads the tuple from ``x``.                              |
+------------------------------------------+----------------------------------------------------------+

.. _class-message:

Class ``message``
-----------------

+--------------------------------------------------+-----------------------------------------------------------------+
| **Observers**                                    |                                                                 |
+==================================================+=================================================================+
| ``bool empty()``                                 | Returns whether this message is empty.                          |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``size_t size()``                                | Returns the size of this message.                               |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``const void* at(size_t p)``                     | Returns a const pointer to the element at position ``p``.       |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``const T& get_as<T>(size_t p)``                 | Returns a const ref. to the element at position ``p``.          |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``bool match_element<T>(size_t p)``              | Returns whether the element at position ``p`` has type ``T``.   |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``bool match_elements<Ts...>()``                 | Returns whether this message has the types ``Ts...``.           |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message drop(size_t n)``                       | Creates a new message with all but the first ``n`` values.      |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message drop_right(size_t n)``                 | Creates a new message with all but the last ``n`` values.       |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message take(size_t n)``                       | Creates a new message from the first ``n`` values.              |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message take_right(size_t n)``                 | Creates a new message from the last ``n`` values.               |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message slice(size_t p, size_t n)``            | Creates a new message from ``[p, p + n)``.                      |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message slice(size_t p, size_t n)``            | Creates a new message from ``[p, p + n)``.                      |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message extract(message_handler)``             | See :ref:`extract`.                                             |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``message extract_opts(...)``                    | See :ref:`extract-opts`.                                        |
+--------------------------------------------------+-----------------------------------------------------------------+
|                                                  |                                                                 |
+--------------------------------------------------+-----------------------------------------------------------------+
| **Modifiers**                                    |                                                                 |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``optional<message> apply(message_handler f)``   | Returns ``f(*this)``.                                           |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``void* mutable_at(size_t p)``                   | Returns a pointer to the element at position p.                 |
+--------------------------------------------------+-----------------------------------------------------------------+
| ``T& get_as_mutable<T>(size_t p)``               | Returns a reference to the element at position p.               |
+--------------------------------------------------+-----------------------------------------------------------------+

.. _class-message_builder:

Class ``message_builder``
-------------------------

+------------------------------------------------------+-------------------------------------------------------+
| **Constructors**                                     |                                                       |
+======================================================+=======================================================+
| ``(void)``                                           | Creates an empty message builder.                     |
+------------------------------------------------------+-------------------------------------------------------+
| ``(Iter first, Iter last)``                          | Adds all elements from range ``[first, last)``.       |
+------------------------------------------------------+-------------------------------------------------------+
|                                                      |                                                       |
+------------------------------------------------------+-------------------------------------------------------+
| **Observers**                                        |                                                       |
+------------------------------------------------------+-------------------------------------------------------+
| ``bool empty()``                                     | Returns whether this message is empty.                |
+------------------------------------------------------+-------------------------------------------------------+
| ``size_t size()``                                    | Returns the size of this message.                     |
+------------------------------------------------------+-------------------------------------------------------+
| ``message to_message(    )``                         | Converts the buffer to an actual message object.      |
+------------------------------------------------------+-------------------------------------------------------+
| ``append(T val)``                                    | Adds ``val`` to the buffer.                           |
+------------------------------------------------------+-------------------------------------------------------+
| ``append(Iter first, Iter last)``                    | Adds all elements from range ``[first, last)``.       |
+------------------------------------------------------+-------------------------------------------------------+
| ``message extract(message_handler)``                 | See :ref:`extract`.                                   |
+------------------------------------------------------+-------------------------------------------------------+
| ``message extract_opts(...)``                        | See :ref:`extract-opts`.                              |
+------------------------------------------------------+-------------------------------------------------------+
|                                                      |                                                       |
+------------------------------------------------------+-------------------------------------------------------+
| **Modifiers**                                        |                                                       |
+------------------------------------------------------+-------------------------------------------------------+
| ``optional<message>`` ``apply(message_handler f)``   | Returns ``f(*this)``.                                 |
+------------------------------------------------------+-------------------------------------------------------+
| ``message move_to_message()``                        | Transfers ownership of its data to the new message.   |
+------------------------------------------------------+-------------------------------------------------------+

.. _extract:

Extracting
----------

The member function ``message::extract`` removes matched elements from a message. x Messages are filtered by repeatedly applying a message handler to the greatest remaining slice, whereas slices are generated in the sequence ``[0, size)``, ``[0, size-1)``, ``...``, ``[1, size-1)``, ``...``, ``[size-1, size)``. Whenever a slice is matched, it is removed from the message and the next slice starts at the same index on the reduced message.

For example:

::

    auto msg = make_message(1, 2.f, 3.f, 4);
    // remove float and integer pairs
    auto msg2 = msg.extract({
      [](float, float) { },
      [](int, int) { }
    });
    assert(msg2 == make_message(1, 4));

Step-by-step explanation:

-  Slice 1: ``(1, 2.f, 3.f, 4)``, no match

-  Slice 2: ``(1, 2.f, 3.f)``, no match

-  Slice 3: ``(1, 2.f)``, no match

-  Slice 4: ``(1)``, no match

-  Slice 5: ``(2.f, 3.f, 4)``, no match

-  Slice 6: ``(2.f, 3.f)``, *match*; new message is ``(1, 4)``

-  Slice 7: ``(4)``, no match

Slice 7 is ``(4)``, i.e., does not contain the first element, because the match on slice 6 occurred at index position 1. The function ``extract`` iterates a message only once, from left to right. The returned message contains the remaining, i.e., unmatched, elements.

.. _extract-opts:

Extracting Command Line Options
-------------------------------

The class ``message`` also contains a convenience interface to ``extract`` for parsing command line options: the member function ``extract_opts``.

::

    int main(int argc, char** argv) {
      uint16_t port;
      string host = "localhost";
      auto res = message_builder(argv + 1, argv + argc).extract_opts({
        {"port,p", "set port", port},
        {"host,H", "set host (default: localhost)", host},
        {"verbose,v", "enable verbose mode"}
      });
      if (! res.error.empty()) {
        // read invalid CLI arguments
        cerr << res.error << endl;
        return 1;
      }
      if (res.opts.count("help") > 0) {
        // CLI arguments contained "-h", "--help", or "-?" (builtin);
        cout << res.helptext << endl;
        return 0;
      }
      if (! res.remainder.empty()) {
        // res.remainder stors all extra arguments that weren't consumed
      }
      if (res.opts.count("verbose") > 0) {
        // enable verbose mode
      }
      // ...
    }

    /*
    Output of ./program_name -h:

    Allowed options:
      -p [--port] arg  : set port
      -H [--host] arg  : set host (default: localhost)
      -v [--verbose]   : enable verbose mode
    */
