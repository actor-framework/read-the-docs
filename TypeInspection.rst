.. _type-inspection:

Type Inspection (Serialization and String Conversion)
=====================================================

CAFis designed with distributed systems in mind. Hence, all message types must be serializable and need a platform-neutral, unique name that is configured at startup (see :ref:`add-custom-message-type`). Using a message type that is not serializable causes a compiler error (see :ref:`unsafe-message-type`). CAF serializes individual elements of a message by using the inspection API. This API allows users to provide code for serialization as well as string conversion with a single free function. The signature for a class ``my_class`` is always as follows:

::

    template <class Inspector>
    typename Inspector::result_type inspect(Inspector& f, my_class& x) {
      return f(...);
    }

The function ``inspect`` passes meta information and data fields to the variadic call operator of the inspector. The following example illustrates an implementation for ``inspect`` for a simple POD struct.

::

    // POD struct foo
    struct foo {
      std::vector<int> a;
      int b;
    };

    // foo needs to be serializable
    template <class Inspector>
    typename Inspector::result_type inspect(Inspector& f, foo& x) {
      return f(meta::type_name("foo"), x.a, x.b);
    }

The inspector recursively inspects all data fields and has builtin support for (1) ``std::tuple``, (2) ``std::pair``, (3) C arrays, (4) any container type with ``x.size()``, ``x.empty()``, ``x.begin()`` and ``x.end()``.

We consciously made the inspect API as generic as possible to allow for extensibility. This allows users to use CAF’s types in other contexts, to implement parsers, etc.

.. _inspector-concept:

Inspector Concept
-----------------

The following concept class shows the requirements for inspectors. The placeholder ``T`` represents any user-defined type. For example, ``error`` when performing I/O operations or some integer type when implementing a hash function.

::

    Inspector {
      using result_type = T;
      
      if (performing a save operation)
        using is_saving = std::true_type;
      else
        using is_loading = std::true_type;
      
      template <class... Ts>
      result_type operator()(Ts&&...);
    }

A saving ``Inspector`` is required to handle constant lvalue reference and rvalue references. A loading ``Inspector`` must only accept mutable lvalue references to data fields, but still allow for constant lvalue references and rvalue references to annotations.

.. _annotations:

Annotations
-----------

Annotations allow users to fine-tune the behavior of inspectors by providing addition meta information about a type. All annotations live in the namespace ``caf::meta`` and derive from ``caf::meta::annotation``. An inspector can query whether a type ``T`` is an annotation with ``caf::meta::is_annotation<T>::value``. Annotations are passed to the call operator of the inspector along with data fields. The following list shows all annotations supported by CAF:

-  ``type_name(n)``: Display type name as ``n`` in human-friendly output (position before data fields).

-  ``hex_formatted()``: Format the following data field in hex format.

-  ``omittable()``: Omit the following data field in human-friendly output.

-  ``omittable_if_empty()``: Omit the following data field if it is empty in human-friendly output.

-  ``omittable_if_none()``: Omit the following data field if it equals ``none`` in human-friendly output.

-  ``save_callback(f)``: Call ``f`` when serializing (position after data fields).

-  ``load_callback(f)``: Call ``f`` after deserializing all data fields (position after data fields).

.. _backwards-and-third-party-compatibility:

Backwards and Third-party Compatibility
---------------------------------------

CAF evaluates common free function other than ``inspect`` in order to simplify users to integrate CAF into existing code bases.

Serializers and deserializers call user-defined ``serialize`` functions. Both types support ``operator&`` as well as ``operator()`` for individual data fields. A ``serialize`` function has priority over ``inspect``.

When converting a user-defined type to a string, CAF calls user-defined ``to_string`` functions and prefers those over ``inspect``.

.. _unsafe-message-type:

Whitelisting Unsafe Message Types
---------------------------------

Message types that are not serializable cause compile time errors when used in actor communication. When using CAFfor concurrency only, this errors can be suppressed by whitelisting types with ``CAF_ALLOW_UNSAFE_MESSAGE_TYPE``. The macro is defined as follows.

::

    #define CAF_ALLOW_UNSAFE_MESSAGE_TYPE(type_name)                               \
      namespace caf {                                                              \
      template <>                                                                  \
      struct allowed_unsafe_message_type<type_name> : std::true_type {};           \
      }

.. _splitting-save-and-load-operations:

Splitting Save and Load Operations
----------------------------------

If loading and storing cannot be implemented in a single function, users can query whether the inspector is loading or storing. For example, consider the following class ``foo`` with getter and setter functions and no public access to its members.

::

    // no friend access for `inspect`
    class foo {
    public:
      foo(int a0 = 0, int b0 = 0) : a_(a0), b_(b0) {
        // nop
      }

      foo(const foo&) = default;
      foo& operator=(const foo&) = default;

      int a() const {
        return a_;
      }

      void set_a(int val) {
        a_ = val;
      }

      int b() const {
        return b_;
      }

      void set_b(int val) {
        b_ = val;
      }

    private:
      int a_;
      int b_;
    };

Since there is no access to the data fields ``a_`` and ``b_`` (and assuming no changes to ``foo`` are possible), we need to split our implementation of ``inspect`` as shown below.

::

    template <class Inspector>
    typename std::enable_if<Inspector::is_saving::value,
                            typename Inspector::result_type>::type
    inspect(Inspector& f, foo& x) {
      return f(meta::type_name("foo"), x.a(), x.b());
    }

    template <class Inspector>
    typename std::enable_if<Inspector::is_loading::value,
                            typename Inspector::result_type>::type
    inspect(Inspector& f, foo& x) {
      struct tmp_t {
        tmp_t(foo& ref) : x_(ref) {
          // nop
        }
        ~tmp_t() {
          // write back to x at scope exit
          x_.set_a(a);
          x_.set_b(b);
        }
        foo& x_;
        int a;
        int b;
      } tmp{x};
      return f(meta::type_name("foo"), tmp.a, tmp.b);
    }
