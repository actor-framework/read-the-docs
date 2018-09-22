.. _utility:

Utility
=======

CAF includes a few utility classes that are likely to be part of C++ eventually (or already are in newer versions of the standard). However, until these classes are part of the standard library on all supported compilers, we unfortunately have to maintain our own implementations.

.. _optional:

Class ``optional``
------------------

Represents a value that may or may not exist.

| ll **Constructors** &  
| ``(T value)`` & Constructs an object with a value.
| ``(none_t = none)`` & Constructs an object without a value.
|   &  
| **Observers** &  
| ``explicit operator bool()`` & Checks whether the object contains a value.
| ``T* operator->()`` & Accesses the contained value.
| ``T& operator*()`` & Accesses the contained value.

.. _class-expected:

Class ``expected``
------------------

Represents the result of a computation that *should* return a value. If no value could be produced, the ``expected<T>`` contains an ``error`` .

| ll **Constructors** &  
| ``(T value)`` & Constructs an object with a value.
| ``(error err)`` & Constructs an object with an error.
|   &  
| **Observers** &  
| ``explicit operator bool()`` & Checks whether the object contains a value.
| ``T* operator->()`` & Accesses the contained value.
| ``T& operator*()`` & Accesses the contained value.
| ``error& error()`` & Accesses the contained error.

.. _constant-unit:

Constant ``unit``
-----------------

The constant ``unit`` of type ``unit_t`` is the equivalent of ``void`` and can be used to initialize ``optional<void>`` and ``expected<void>``.

.. _constant-none:

Constant ``none``
-----------------

The constant ``none`` of type ``none_t`` can be used to initialize an ``optional<T>`` to represent “nothing”.
