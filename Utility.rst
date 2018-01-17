.. raw:: latex

   \definecolor{lightgrey}{rgb}{0.9,0.9,0.9}

.. raw:: latex

   \definecolor{lightblue}{rgb}{0,0,1}

.. raw:: latex

   \definecolor{grey}{rgb}{0.5,0.5,0.5}

.. raw:: latex

   \definecolor{blue}{rgb}{0,0,1}

.. raw:: latex

   \definecolor{violet}{rgb}{0.5,0,0.5}

.. raw:: latex

   \definecolor{darkred}{rgb}{0.5,0,0}

.. raw:: latex

   \definecolor{darkblue}{rgb}{0,0,0.5}

.. raw:: latex

   \definecolor{darkgreen}{rgb}{0,0.5,0}

.. _utility:

Utility
=======

CAF includes a few utility classes that are likely to be part of C++ eventually (or already are in newer versions of the standard). However, until these classes are part of the standard library on all supported compilers, we unfortunately have to maintain our own implementations.

.. _optional:

Class ``optional``
------------------

Represents a value that may or may not exist.

.. raw:: latex

   \small

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

Represents the result of a computation that *should* return a value. If no value could be produced, the ``expected<T>`` contains an ``error`` (see § `:ref:`error` <#error>`__).

.. raw:: latex

   \small

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
