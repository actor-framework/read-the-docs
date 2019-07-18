.. _error:

Errors
======

Errors in CAF have a code and a category, similar to ``std::error_code`` and ``std::error_condition``. Unlike its counterparts from the C++ standard library, ``error`` is plattform-neutral and serializable. Instead of using category singletons, CAF stores categories as atoms . Errors can also include a message to provide additional context information.

.. _class-interface:

Class Interface
---------------

======================================== ==================================================================
**Constructors**                          
======================================== ==================================================================
``(Enum x)``                             Construct error by calling ``make_error(x)``
``(uint8_t x, atom_value y)``            Construct error with code ``x`` and category ``y``
``(uint8_t x, atom_value y, message z)`` Construct error with code ``x``, category ``y``, and context ``z``
                                          
**Observers**                             
``uint8_t code()``                       Returns the error code
``atom_value category()``                Returns the error category
``message context()``                    Returns additional context information
``explicit operator bool()``             Returns ``code() != 0``
======================================== ==================================================================

.. _custom-error:

Add Custom Error Categories
---------------------------

Adding custom error categories requires three steps: (1) declare an enum class of type ``uint8_t`` with the first value starting at 1, (2) implement a free function ``make_error`` that converts the enum to an ``error`` object, (3) add the custom category to the actor system with a render function. The last step is optional to allow users to retrieve a better string representation from ``system.render(x)`` than ``to_string(x)`` can offer. Note that any error code with value 0 is interpreted as *not-an-error*. The following example adds a custom error category by performing the first two steps.

The implementation of ``to_string(error)`` is unable to call string conversions for custom error categories. Hence, ``to_string(make_error(math_error::division_by_zero))`` returns ``"error(1, math)"``.

The following code adds a rendering function to the actor system to provide a more satisfactory string conversion.

With the custom rendering function, ``system.render(make_error(math_error::division_by_zero))`` returns ``"math_error(division_by_zero)"``.

.. _sec:

System Error Codes
------------------

System Error Codes (SECs) use the error category ``"system"``. They represent errors in the actor system or one of its modules and are defined as follows.

.. _exit-reason:

Default Exit Reasons
--------------------

CAF uses the error category ``"exit"`` for default exit reasons. These errors are usually fail states set by the actor system itself. The two exceptions are ``exit_reason::user_shutdown`` and ``exit_reason::kill``. The former is used in CAF to signalize orderly, user-requested shutdown and can be used by programmers in the same way. The latter terminates an actor unconditionally when used in ``send_exit``, even if the default handler for exit messages  is overridden.
